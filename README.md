# Stable Diffusion Instância SPOT EC2 AWS   

## Começe por aqui!

Pedido de aumento de quota na AWS 

1. Vá até [AWS Service Quota](https://us-east-1.console.aws.amazon.com/servicequotas/home/services/ec2/quotas) painel de controle (verificar a região).
3. Procurar por `All G and VT Spot Instance Requests` e clicar nele.
4. Clique em `Aumentar a cota da conta` e introduza o valor `4` na caixa de entrada.
5. Clique em "Solicitar" para apresentar o seu pedido de aumento de cota.


Use o [RUNME](https://github.com/stateful/runme) para facilitar a execução dos snippets a seguir.

### Lançar Spot

#### Criar o pedido de instância spot (que criará a instância após alguns segundos)

```bash {name=criar-instancia}
export PUBLIC_KEY_PATH="$HOME/.ssh/id_rsa.pub"
export INSTALL_AUTOMATIC1111="true"
export INSTALL_INVOKEAI="false"
export GUI_TO_START="automatic1111"
export EBS_SIZE="80"

aws ec2 import-key-pair --key-name stable-diffusion-aws --public-key-material fileb://${PUBLIC_KEY_PATH} --tag-specifications 'ResourceType=key-pair,Tags=[{Key=creator,Value=stable-diffusion-aws}]'

# Obtem a última imagem Debian 12 Linux
export AMI_ID=$(aws ec2 describe-images --owners 136693071363 --query "sort_by(Images, &CreationDate)[-1].ImageId" --filters "Name=name,Values=debian-12-amd64-*" | jq -r .)

export DEFAULT_VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
export SG_ID=$(aws ec2 create-security-group --group-name SSH-Only --description "Allow SSH from anywhere" --vpc-id $DEFAULT_VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 create-tags --resources $SG_ID --tags Key=creator,Value=stable-diffusion-aws

aws ec2 run-instances \
    --no-cli-pager \
    --image-id $AMI_ID \
    --instance-type g4dn.xlarge \
    --key-name stable-diffusion-aws \
    --security-group-ids $SG_ID \
    --block-device-mappings "DeviceName=/dev/xvda,Ebs={VolumeSize=${EBS_SIZE},VolumeType=gp3}" \
    --user-data file://setup.sh \
    --metadata-options "InstanceMetadataTags=enabled" \
    --tag-specifications "ResourceType=spot-instances-request,Tags=[{Key=creator,Value=stable-diffusion-aws}]" "ResourceType=instance,Tags=[{Key=INSTALL_AUTOMATIC1111,Value=$INSTALL_AUTOMATIC1111},{Key=INSTALL_INVOKEAI,Value=$INSTALL_INVOKEAI},{Key=GUI_TO_START,Value=$GUI_TO_START}]" \
    --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.20,SpotInstanceType=persistent,InstanceInterruptionBehavior=stop}'

```

#### Criar um alarme para parar a instância após 15 minutos de inatividade (opcional)

```bash {name=criar-alarme, promptEnv=false}
export SPOT_INSTANCE_REQUEST="$(aws ec2 describe-spot-instance-requests --filters 'Name=tag:creator,Values=stable-diffusion-aws' 'Name=state,Values=active,open' | jq -r '.SpotInstanceRequests[].SpotInstanceRequestId')"
export INSTANCE_ID="$(aws ec2 describe-spot-instance-requests --spot-instance-request-ids $SPOT_INSTANCE_REQUEST | jq -r '.SpotInstanceRequests[].InstanceId')"

aws cloudwatch put-metric-alarm \
    --alarm-name stable-diffusion-aws-stop-when-idle \
    --namespace AWS/EC2 \
    --metric-name CPUUtilization \
    --statistic Maximum \
    --period 300  \
    --evaluation-periods 3 \
    --threshold 5 \
    --comparison-operator LessThanThreshold \
    --unit Percent \
    --dimensions "Name=InstanceId,Value=$INSTANCE_ID" \
    --alarm-actions arn:aws:automate:$AWS_REGION:ec2:stop
```

#### Conectar

```bash {name=conectar-via-ssh, promptEnv=false}
export SPOT_INSTANCE_REQUEST="$(aws ec2 describe-spot-instance-requests --filters 'Name=tag:creator,Values=stable-diffusion-aws' 'Name=state,Values=active,open' | jq -r '.SpotInstanceRequests[].SpotInstanceRequestId')"
export INSTANCE_ID="$(aws ec2 describe-spot-instance-requests --spot-instance-request-ids $SPOT_INSTANCE_REQUEST | jq -r '.SpotInstanceRequests[].InstanceId')"
export PUBLIC_IP="$(aws ec2 describe-instances --instance-id $INSTANCE_ID | jq -r '.Reservations[].Instances[].PublicIpAddress')"
ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -L7860:localhost:7860 -L9090:localhost:9090 admin@$PUBLIC_IP

# Wait about 10 minutes from the first creation

# Open http://localhost:7860 or http://localhost:9090
```

### Gestão do ciclo de Trabalho

#### Parar Instância

```bash {name=parar-instancia, promptEnv=false}
export SPOT_INSTANCE_REQUEST="$(aws ec2 describe-spot-instance-requests --filters 'Name=tag:creator,Values=stable-diffusion-aws' 'Name=state,Values=active,open' | jq -r '.SpotInstanceRequests[].SpotInstanceRequestId')"
export INSTANCE_ID="$(aws ec2 describe-spot-instance-requests --spot-instance-request-ids $SPOT_INSTANCE_REQUEST | jq -r '.SpotInstanceRequests[].InstanceId')"
aws ec2 stop-instances --instance-ids $INSTANCE_ID
```

#### Iniciar Instância

```bash {name=iniciar-instancia, promptEnv=false}
export SPOT_INSTANCE_REQUEST="$(aws ec2 describe-spot-instance-requests --filters 'Name=tag:creator,Values=stable-diffusion-aws' 'Name=state,Values=disabled' | jq -r '.SpotInstanceRequests[].SpotInstanceRequestId')"
export INSTANCE_ID="$(aws ec2 describe-spot-instance-requests --spot-instance-request-ids $SPOT_INSTANCE_REQUEST | jq -r '.SpotInstanceRequests[].InstanceId')"
aws ec2 start-instances --instance-ids $INSTANCE_ID
```

#### Deletar Instância

```bash {name=limpar-tudo, promptEnv=false}
export SPOT_INSTANCE_REQUEST="$(aws ec2 describe-spot-instance-requests --filters 'Name=tag:creator,Values=stable-diffusion-aws' 'Name=state,Values=active,open,disabled' | jq -r '.SpotInstanceRequests[].SpotInstanceRequestId')"
[[ -n $SPOT_INSTANCE_REQUEST ]] && export INSTANCE_ID="$(aws ec2 describe-spot-instance-requests --spot-instance-request-ids $SPOT_INSTANCE_REQUEST | jq -r '.SpotInstanceRequests[].InstanceId')"
export SG_ID="$(aws ec2 describe-security-groups --filters 'Name=tag:creator,Values=stable-diffusion-aws' --query 'SecurityGroups[*].GroupId' --output text)"
export KEY_PAIR_NAME="$(aws ec2 describe-key-pairs --filters 'Name=tag:creator,Values=stable-diffusion-aws' --query 'KeyPairs[0].KeyName' --output text)"
[[ -n $SPOT_INSTANCE_REQUEST ]] && aws ec2 cancel-spot-instance-requests --spot-instance-request-ids $SPOT_INSTANCE_REQUEST
if [[ -n $INSTANCE_ID ]]
then
    VPC_ID=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[0].Instances[0].VpcId' --output text)
    DEFAULT_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=default" --query 'SecurityGroups[0].GroupId' --output text)
    aws ec2 modify-instance-attribute --instance-id $INSTANCE_ID --groups $DEFAULT_SG_ID
    aws ec2 terminate-instances --instance-ids $INSTANCE_ID
fi

[[ -n $KEY_PAIR_NAME ]] && aws ec2 delete-key-pair --key-name $KEY_PAIR_NAME
[[ -n $SG_ID ]] && aws ec2 delete-security-group --group-id $SG_ID
aws cloudwatch delete-alarms --alarm-names stable-diffusion-aws-stop-when-idle
```

## Explicação completa

Este repositório facilita a execução da sua própria instância Spot do Stable Diffusion no EC2 AWS. Existem duas opções para o frontend; a primeira é a GUI em https://github.com/AUTOMATIC1111/stable-diffusion-webui, e a segunda é https://github.com/invoke-ai/InvokeAI. Por padrão, somente a AUTOMATIC1111 é instalado. Mas você pode optar por instalar a INVOKEAI ao iniciar. Lembre-se,  não existe memória RAM suficiente para rodar ambos ao mesmo tempo, uma vez que o carregamento de modelos + geração de imagens ocupará um pouco mais de 16GB de RAM. Existem variáveis de ambiente no início do setup.sh que podem ser usadas para definir quais são instaladas e/ou iniciadas. Os serviços do Systemd são instalados para ambos, e eles podem ser iniciados ou parados em tempo de execução livremente. Os nomes são `sdwebgui.service` e `invokeai.service`. 

A secção "Lançar Spot" contém snippets para criar um pedido de instância spot que irá lançar uma instância spot. O preço de uma instância sob demanda g4dn.xlarge é $0,52/hora, mas uma instância spot flutua atualmente em volta de $0,17, o que representa uma poupança de 65%. Estas instruções definem um limite de preço de $0,20; se precisar de uma maior viabilidade, pode remover `MaxPrice=0.20,` e isso permitirá que custe até o preço total do sob demanda.

Esta instância spot pode ser parada e iniciada como uma instância normal. Quando parada, o único custo é de US$ 0,40/mês para o volume EBS. Ao remover todos os vestígios disso, observe que encerrar a instância fará com que o SpotInstanceRequest inicie uma nova instância, mas, por outro lado, cancelar o SpotInstanceRequest não encerrará automaticamente as instâncias que ele gerou. Assim, o SpotInstanceRequest deve ser cancelado primeiro e, em seguida, a instância deve ser explicitamente encerrada.

Há aproximadamente 50GB livres na partição raiz. Isso deve ser suficiente para a operação básica, mas se você precisar de mais espaço temporariamente, você pode usar `/mnt/ephemeral`, que é um volume de instância de 125GB (115 GB). É um SSD de alto desempenho, mas será limpo em cada paragem/início da instância EC2. Ele também contém um swapfile de 8GB.

Para poupar custos, a instância será automaticamente encerrada se a utilização da CPU (amostrada a cada 5 minutos) for inferior a 20% durante 3 verificações consecutivas. Isso pode ser ignorado, se desejado.

Lembre-se que não existe nenhum tipo de proteção para acessar o GUI. Por isso, crie um túnel ssh e ligue-se através de http://localhost:7860 para automatic1111 ou http://localhost:9090 para Invoke-AI.

Algumas partes do script são baseadas em https://github.com/mikeage/stable-diffusion-aws
