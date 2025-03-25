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
1) Retirar criptografia disco EBS:  <br>
Utilizando-se de uma instância Linux, Attach o disco criptografado de origem como /dev/sdb e outro disco novo vazio(Mesmo tamanho) como /dev/sdc:  <br>
<br>
![IMAGE02](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image02.png) <br>
![IMAGE03](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image03.png) <br>
![IMAGE04](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image04.png) <br>
<br>
2) Execute o comando dd para efetuar a cópia de um disco para o outro: <br> <br>
dd if=/dev/sdb of=/dev/sdc bs=4096 status=progress <br> <br>
Esse processo irá demorar. Monitore para até quando finalizar a cópia. <br>
Finalizada a cópia teremos o seguinte resultado, com o disco sem criptografia: <br>
<br>
![IMAGE05](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image05.png) <br>
<br>
3) Criar instância com mesmo SO de origem e Attach disco sem criptografia. <br>
<br>
4) Criar Bucket para geração de arquivo OVA <br>
<br>
![IMAGE06](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image06.png)  <br>
<br>
https://docs.aws.amazon.com/vm-import/latest/userguide/vmexport-prerequisites.html <br>
<br>
5) Exportar instancia para Bucket <br> <br>
aws ec2 create-instance-export-task --instance-id i-06d09e04614 --description "Export-To-OCI" --target-environment vmware --export-to-s3-task DiskImageFormat=vmdk,ContainerFormat=ova,S3Bucket=migrateawstooci,S3Prefix=vms/ <br>
<br>
![IMAGE07](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image07.png) <br>
<br>
aws ec2 describe-export-tasks --export-task-ids export-i-223be915b76e7t <br>
aws s3 ls migrateawstooci <br>
<br>
6) Copiar arquivo OVA gerado no Bucket para uma pasta local: <br> <br>
aws s3 cp s3://migrateawstooci/instance.ova /mnt/dados <br>
<br>
7) Converter OVA para VMDK <br>
![IMAGE08](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image08.png) <br>
<br>
8) Realizar cópia do arquivo VMDK para OCI <br> <br>
oci os object put -bn Bucket-Oracle --file /mnt/dados/instance-disk-1.vmdk <br>
<br>
9) Criar Image a partir do arquivo VMDK no Bucket <br>
![IMAGE09](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image09.png) <br>
<br>
10) Criar instancia a partir da imagem criada <br>
![IMAGE10](https://github.com/fernandomxm/Migrate-EC2-AWS-to-OCI/blob/main/image10.png) <br>
<br>
Migrar instâncias e volumes está sujeito às estas outras limitações: <br>
<br>
- Você não pode exportar uma VM se ela contiver software de terceiros fornecido pela AWS. Por exemplo, o VM Export não pode exportar instâncias do Windows ou SQL Server, ou qualquer instância criada a partir de uma imagem no AWS Marketplace. <br>
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
