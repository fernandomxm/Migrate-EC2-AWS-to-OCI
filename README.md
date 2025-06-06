# Migrate-EC2-AWS-to-OCI
Migrate EC2 AWS to OCI <br>
 <br>
Caso a instância que pretende migrar de AWS para OCI tiver o seu volume <b> EBS criptografado </b> , siga a partir do passo 1. <br>
Caso o volume <b> EBS não seja criptografado </b> , siga a partir do passo 4. <br>
<br>
![IMAGE01](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image01.png) <br>
<br>
https://docs.aws.amazon.com/vm-import/latest/userguide/limits-image-export.html  <br>
<br>
Para evitar também o problema de não carregamento de drivers de disco de boot: <br>
![Erro_Boot_Windows](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/Erro_Boot_Windows.png) <br>
Instalar e reiniciar o servidor, antes de gerar o arquivo OVA, o pacote Oracle Windows VirtIO drivers: <br>
![Oracle_VirtIO_Drivers](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/Oracle_VirtIO_Drivers.png) <br> 
https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/importingcustomimagewindows.htm <br>
https://docs.oracle.com/en/operating-systems/oracle-linux/kvm-virtio/kvm-virtio-DownloadingtheOracleVirtIODriversforMicrosoftWindows.html <br> <br>
1) Retirar criptografia disco EBS:  <br>
Utilizando-se de uma instância Linux, Attach o disco criptografado de origem como /dev/sdb e outro disco novo vazio(Mesmo tamanho) como /dev/sdc:  <br>

![IMAGE02](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image02.png) <br>

![IMAGE03](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image03.png) <br>

![IMAGE04](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image04.png) <br>

2) Execute o comando dd para efetuar a cópia de um disco para o outro: <br> <br>
<b> dd if=/dev/sdb of=/dev/sdc bs=4096 status=progress </b> <br> <br>
Esse processo irá demorar. Monitore para até quando finalizar a cópia. <br>
Finalizada a cópia teremos o seguinte resultado, com o disco sem criptografia: <br>

![IMAGE05](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image05.png) <br>

3) Criar instância com mesmo SO de origem e Attach disco sem criptografia. <br>
<br>
4) Criar Bucket para geração de arquivo OVA <br>

![IMAGE06](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image06.png)  <br>

https://docs.aws.amazon.com/vm-import/latest/userguide/vmexport-prerequisites.html <br>
<br>
5) Exportar instancia para Bucket AWS <br> <br>
<b> aws ec2 create-instance-export-task --instance-id i-06d09e04614 --description "Export-To-OCI" --target-environment vmware --export-to-s3-task DiskImageFormat=vmdk,ContainerFormat=ova,S3Bucket=migrateawstooci,S3Prefix=vms/  </b> <br>
<br>
Opcionalmente pode ser utilizado esse comando: <br>
aws ec2 export-image --image-id ami-016e07 --disk-image-format VMDK --s3-export-location S3Bucket=migrateawstooci <br>
<br>
![IMAGE07](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image07.png) <br>
<br>
Caso tenha algum erro relacionado a AMI, o disco pode ser atachado em um linux e executados os comandos abaixo: <br>
dd if=/dev/xvdb1 of=~/ec2-disk.img bs=4M status=progress conv=sync,noerror <br>
qemu-img convert -f raw ec2-disk.img -O vmdk ec2-disk.vmdk <br>
Pular para passo 8 <br>
<br>
<b> aws ec2 describe-export-tasks --export-task-ids export-i-223be915b76e7t <br>
![create_ova](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/Create_OVA.png) <br>
aws s3 ls migrateawstooci  </b> <br>
<br>
6) Copiar arquivo OVA gerado no Bucket para uma pasta local: <br> <br>
<b> aws s3 cp s3://migrateawstooci/instance.ova /mnt/dados  </b> <br>
<br>
7) Extrair o arquivo VMDK do arquivo OVA <br>
![IMAGE08](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image08.png) <br>
<br>
8) Realizar cópia do arquivo VMDK para OCI <br> <br>
<b> oci os object put -bn Bucket-Oracle --file /mnt/dados/instance-disk-1.vmdk  </b> <br>
<br>
9) Criar Image a partir do arquivo VMDK no Bucket <br>
![IMAGE09](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image09.png) <br>
<br>
10) Criar instancia a partir da imagem criada <br>
![IMAGE10](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image10.png) <br>
<br>
Migrar instâncias e volumes está sujeito a estas limitações: <br>
<br>
- Você não pode exportar uma VM se ela contiver software de terceiros fornecido pela AWS. Por exemplo, o <u><strong> VM Export não pode exportar instâncias do Windows ou SQL Server </strong></u>, ou qualquer instância criada a partir de uma imagem no AWS Marketplace. <br>
- Você deve exportar suas instâncias e volumes para um dos seguintes formatos de imagem que seu ambiente de virtualização suporta: <br>
Open Virtual Appliance (OVA) <br>
Virtual Hard Disk (VHD) <br>
Stream-optimized ESX Virtual Machine Disk (VMDK) <br>
- Você não pode exportar volumes de dados do Amazon EBS. <br>
- Você não pode exportar uma instância que tenha mais de um disco virtual. <br>
- Você não pode exportar uma instância que tenha mais de uma interface de rede. <br>
- Você não pode exportar uma instância do Amazon EC2 se você a compartilhou de outra conta da AWS. <br>
- Você não pode ter mais de cinco tarefas de exportação por região em andamento ao mesmo tempo. <br>
- VMs com volumes maiores que 1 TiB não são suportadas. <br>
- Você pode exportar um volume para um bucket S3 não criptografado ou para um bucket criptografado usando SSE-S3. <br>
Documentação oficial: https://docs.aws.amazon.com/vm-import/latest/userguide/vmexport.html <br>
