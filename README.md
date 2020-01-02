# Hidden Volume – Dmcrypt

Criptografar um volume pode ser muito útil pois só vc tem acesso e pode guardar o que quiser lá dentro. Mas tem certas situações onde isso torna inútil. Imagine que você tenha lá o seu dispositivo criptografado e cheguem dois caras na sua porta e ameaçam afogar seu porquinho da índia no vazo sanitário se você não der a senha do seu dispositivo. Em um caso como esse, não há como negar a sua senha, você não viveria sem o Tom, o porquinho. Por isso um método muito bom seria você ter, dentro desse dispositivo, um volume oculto criptografado a onde ficaria os dados sigilosos. Seria como um cofre dentro de outro cofre, sacou? vamos lá.

## CRIANDO O CONTAINER EXTERNO

Bom, vamos criar agora nosso container externo chamado cofre.img com o tamanho de 100MB e adicionar um endereço de loop. O endereço de loop vai apontar para o nosso img criada. Então /dev/loop0 = cofre.img
```sh
$ dd if=/dev/zero of=cofre.img bs=10M count=10
$ losetup /dev/loop0 cofre.img
```
Vamos criptografar ele com luks simples. Aqui, não é importante que você tenha uma senha forte, pois aqui será apenas uma faixada para o nosso cofre verdadeiro.
Vamos também abrir ele com o nome de cofre1. Esse dispositivo irá ficar localizado em /dev/mapper/cofre1.
```sh
$ cryptsetup luksFormat /dev/loop0
$ cryptsetup luksOpen /dev/loop0 cofre1
```
aberto o nosso container, agora vamos enche-lo de dados aleatórios para ajudar na camuflagem do nosso volume oculto. Por que não zeros? porque com zeros seria fácil perceber que tem um amontoado de dados ali no meio.
```sh
$ dd if=/dev/urandom of=/dev/mapper/cofre1
```
Agora vamos formatar um sistema de arquivos para o dispositivo. Vou usar FAT. Sinta-se livre para escolher o seu.
```sh
$ mkfs.vfat /dev/mapper/cofre1
```
Agora, vamos montar cofre1 e  jogar arquivos aleatorios para lá, só para deixar o volume externo de fachada mais convicente. Vou jogar uns books que tenho aqui só para exemplo.

É importante lembrar que, é extremamente perigoso escrever dados no volume externo após criar o volume oculto, há grandes chances de sobreescrever os dados dele. É recomendado que você grave os arquivos  que você quer no volume externo antes de criar o volume oculto.
```sh
$ mount /dev/mapper/cofre1 mount_pointer/
 
$ ls -l mount_pointer/
-rwxr-xr-x 1 root root 1597605 Dec 10 18:40 aprenda_computacao_com_python3_k.pdf
-rwxr-xr-x 1 root root 7314537 Dec 10 18:40 Botnets.pdf
-rwxr-xr-x 1 root root 173437 Dec 10 18:40 canivete-shell.pdf
```
Pronto, criamos nosso volume de faixada. Agora seu porquinho da índia Tom já tem uma chance de viver. Desmonte-o e vamos para o proximo passo

## CRIANDO O VOLUME OCULTO CRIPTOGRAFADO

Bom, para criarmos nosso volume oculto, não usaremos LUKS. LUKS, por padrão, possui um header, onde possui informações sobre o volume como hash da senha, versão etc., Isso estragaria nossa camuflagem. Continuaria criptografado, porém não escondido, o que não é o nosso objetivo. Então, ao invés de usar o LUKS, vamos usar em modo Plain. O modo Plain, ao contrário do LUKS, é puro, então ele deixa que a gente mexa com blocos brutos sem nenhuma outra interferência. Portanto, ele não avisará se você colocar a senha ou o bloco incorreto, ele apenas irá aplicar o algoritmo.

O mais importante aqui é o offset, o setor de onde irá começar a leitura. Cada setor tem o tamanho de 512 bytes. Vamos olhar quanto setores tem no total do nosso container de 100MB usando o fdisk.
```sh
$ fdisk -l /dev/mapper/cofre1
Disk /dev/mapper/cofre1: <strong>98 MiB</strong>, 102760448 bytes, <strong>200704 sectors</strong>
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000
```
Nós temos o total de 200704 setores, como vamos colocar o nosso volume no meio, o valor do offset vai ser de 100352. 200704/2 = 100352.

Ele irá pedir uma senha e criará nosso volume oculto chamado de cofre2. Lembrando que, o modo plain não tem verificação de senha.

```sh
$ sudo cryptsetup --cipher=aes-xts-plain64 --offset=100352 plainOpen /dev/mapper/cofre1 cofre2
```
Pronto, agora temos nosso volume oculto criptografado. Agora vamos apenas formatar um sistema de arquivos para ele.
```sh
$ mkfs.vfat /dev/mapper/cofre2
```

Yeah, está feito, salvamos o Tom porra. agora é só montar ele e guardar suas tralhas. Lembrando que você tem que saber tanto a senha quanto o offset e a cifra, todos devem estar junto para descriptografar corretamente.

Agora vamos testar.
```sh
$ cryptsetup luksOpen /dev/loop0 cofre1
Enter passphrase for /dev/loop0:
$ mount /dev/mapper/cofre1 mount_pointer/
$ ls mount_pointer
aprenda_computacao_com_python3_k.pdf Botnets.pdf canivete-shell.pdf
$ umount mount_pointer
$ sudo cryptsetup --cipher=aes-xts-plain64 --offset=100352 plainOpen /dev/mapper/cofre1 cofre2
Enter passphrase for /dev/mapper/cofre1:
$ mount /dev/mapper/cofre2 mount_pointer/
$ ls mount_pointer/
HIDDEN receita_do_tom.txt
```
## CONCLUSÃO

Este método é muito bom para esconder volumes para situações que a senha é inegável, porém, por conta de ser bruto, tem mais riscos de você fazer merda e acabar perdendo esses dados. De qualquer forma, salvamos a vida do Tom porra!

### Fonte
https://www.linuxvoice.com/hidden-encrypted-volumes-keep-data-safe-and-secret/

Deus te abençõe. Ou não, eu sou ateu.
