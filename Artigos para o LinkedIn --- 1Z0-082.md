

### POST 1 **SPFILE e PFILE: Como Gerenciar e Recuperar os Arquivos de Inicialização do Oracle Database**


Durante o processo de `STARTUP` no Oracle Database, dois tipos de arquivos de parâmetros de inicialização são fundamentais para que a instância se torne operacional: o SPFILE e o PFILE.

**SPFILE (Server Parameter File):**

- É um arquivo binário com capacidade _read/write_ responsável por armazenar os parâmetros de inicialização da instância.
- Alterações feitas nos parâmetros enquanto a instância está em execução podem ser aplicadas dinamicamente e são persistentes, ou seja, serão mantidas após o encerramento e reinício da instância.
- Por ser um arquivo binário, o SPFILE não pode ser editado diretamente em um editor de texto; as modificações devem ser feitas através do comando SQL:
```sql
ALTER SYSTEM SET parameter_name = value [SCOPE=MEMORY|SPFILE|BOTH];
```

**PFILE (Parameter File):**

- É um arquivo texto, com acesso _read-only_ pelo Oracle Database durante a inicialização.
- O banco de dados apenas lê este arquivo para carregar seus parâmetros; ele não realiza escrita no PFILE.
- O PFILE deve ser editado manualmente pelo DBA, em casos onde não se utiliza SPFILE.

Quando o comando `STARTUP` é executado **sem especificar explicitamente um PFILE**, o Oracle, por padrão, tenta encontrar e carregar o **SPFILE** para iniciar a instância. Caso o SPFILE não esteja disponível, o banco de dados inicializará utilizando o PFILE

Localização padrão dos arquivos de inicialização:

```
ls -lh $ORACLE_HOME/dbs;
```

![[Pasted image 20260228200642.png]]

Durante a manutenção dos parâmetros do banco de dados, o DBA, por precaução, criou uma cópia do PFILE para garantir uma restauração rápida caso ocorram falhas durante as alterações.

Criação de um diretório alternativo, fora do caminho padrão dos arquivos de inicialização, dentro da pasta `/home/oracle`:
```
mkdir /home/oracle/backup_pfile
```

Criação de uma cópia do arquivo PFILE:
```sql
CREATE PFILE='/home/oracle/backup_pfile/pfile_bkp.ora' FROM SPFILE; 
```
![[Pasted image 20260228203325.png]]


Visualizando o arquivo  no diretório criado:
```
ls -lh /home/oracle/backup_pfile;
```
![[Pasted image 20260228224742.png]]

Caso o caminho não seja especificado, o Oracle salva o arquivo PFILE no diretório padrão, que normalmente é `$ORACLE_HOME/dbs`.

Durante a alteração dos parâmetros do banco de dados, o DBA removeu acidentalmente os arquivos responsáveis pela inicialização da instância.
```
 cd $ORACLE_HOME/dbs;
 ls -lh;
 rm -f initorcl.ora;
 rm -f spfileorcl.ora;
 ls -lh;
```

![[Pasted image 20260228204806.png]]

Porém, apesar da remoção dos arquivos, a instância continuou rodando normalmente; o problema só foi detectado após a tentativa de alterar um parâmetro no SPFILE.
```
ALTER SYSTEM SET processes=500 SCOPE=SPFILE;
```
![[Pasted image 20260228205332.png]]


Acreditando que se tratava de um travamento na instância, optou-se pela reinicialização imediata na tentativa de resolver o problema:
```
SHUTDOWN IMMEDIATE;
STARTUP;
```

![[Pasted image 20260228205720.png]]


O problema foi ainda maior, pois a instância não podia ser aberta nem o parâmetro alterado.

**Solução:** iniciar a instância utilizando a cópia de segurança do PFILE criada antes das manutenções.

```
STARTUP PFILE='/home/oracle/backup_pfile/pfile_bkp.ora'
```
![[Pasted image 20260228210130.png]]

Instância recuperada e realizada uma nova tentativa de alterar o parâmetro `processes`

```
ALTER SYSTEM SET processes=500 SCOPE=SPFILE;
```

![[Pasted image 20260228210500.png]]

Um novo erro foi identificado: embora a instância tenha sido recuperada através do PFILE, o SPFILE permanece fora de operação. Para solucionar esse problema, existem duas alternativas:

1. Recuperar o SPFILE a partir dos dados em memória;
    
2. Criar o SPFILE novamente utilizando a cópia do próprio PFILE em uso.

```
CREATE SPFILE FROM PFILE='/home/oracle/backup_pfile/pfile_bkp.ora'
```

Em nosso ambiente, recuperaremos o spfile, a partir de dados da memoria:
```
CREATE SPFILE FROM MEMORY;
```
![[Pasted image 20260228210827.png]]



Conferindo o diretório padrão:
```
ls -lh $ORACLE_HOME/dbs;
```

![[Pasted image 20260228211103.png]]

Como o spfile é o arquivo padrão de inicialização, reiniciando a instancia, a mesma utilizara esse arquivo:
```
SHUTDOWN IMMEDIATE;
STARTUP;
```
![[Pasted image 20260228211404.png]]


Novamente a tentativa de alterar  o parametro process

```
ALTER SYSTEM SET processes=500 SCOPE=SPFILE;
```
![[Pasted image 20260228211530.png]]


Alteração realizada com sucesso! O SPFILE foi recuperado e está em uso. A partir dele, podemos gerar um PFILE para manter ambos os arquivos atualizados no diretório padrão.

```
CREATE PFILE FROM SPFILE;
```
![[Pasted image 20260228211723.png]]

Ao verificar o diretório padrão, ambos os arquivos estarão presentes:
```
ls -lh $ORACLE_HOME/dbs;
```

![[Pasted image 20260228211916.png]]

**Conclusão:**  
Ao realizar alterações nos parâmetros de inicialização do banco de dados, existe o risco de configurar incorretamente a instância, o que pode resultar em sua indisponibilidade. Por isso, é fundamental manter uma cópia de segurança atualizada do arquivo PFILE antes de qualquer modificação. Essa prática simples e preventiva garante uma via rápida e segura para a recuperação do ambiente, minimizando o tempo de inatividade e o impacto operacional.








### POST 2  **Gerenciando Control Files**

É um arquivo binário que registra a estrutura física do banco de dados; 



==**O control file inclui:** ==
***O nome do banco de dados;*** 
***Nomes e localização de data files e redo log files;*** 
***O timestamp de criação do banco;*** 
***A sequência do log atual;*** 
***Informações de checkpoint;***

O control file deve estar disponível para gravação pelo servidor Oracle Database sempre que o banco de dados estiver aberto; 

Sem o control file , o banco de dados não pode ser montado e a recuperação se torna mais difícil; 

O control file de um banco de dados Oracle é criado ao mesmo tempo que o banco de dados;

**Localização dos Control Files** 

O parâmetro de inicialização define os locais dos control files; 

A instância reconhece e abre todos os arquivos listados durante a fase de MOUNT do startup; 

Ela também escreve e mantém estes control files atualizados durante a operação do banco de dados;



**Backup dos Control Files** 
É muito importante que os control files tenham backup; 

Os backups devem ser feitos todas as vezes que alteramos a ==estrutura física== do banco de dados, com operações como: 

Adicionar, descartar ou renomear data files; 

Adicionar ou eliminar uma tablespace ou alterar o estado de leitura/gravação da tablespace; 

Adicionar ou eliminar arquivos ou grupos de redo log; 

O RMAN conta com a configuração de CONTROLFILE AUTOBACKUP, que faz um backup do controlfile a cada vez que um backup é feito, e caso o banco esteja em modo archivelog, também identifica as alterações na estrutura física, e faz um backup automático do control files após elas ocorrerem;

Podemos fazer o backup via SQL*Plus de duas maneiras: 

Fazer o backup do control file como uma cópia binária: 
```sql
ALTER DATABASE BACKUP CONTROLFILE TO '/home/oracle/control_file.bkp';
``` 

Criar um script SQL de recriação do control file: 
```sql
ALTER DATABASE BACKUP CONTROLFILE TO TRACE; 
```

Esse modo escreve os comandos SQL necessários para recriar um control file do zero. Devemos verificar o alert log para encontrar a localização onde o banco salvou o control file;

Recuperação dos Control Files

Para recuperar o control file perdido ou corrompido utilizando uma cópia binária, basta copiarmos o backup para a localização do control file com problema, com o banco de dados em modo shutdown: 

```
$ cp /home/oracle/control_file.bkp /u02/oradata/ORCL/control01.ctl 
```

Depois da cópia feita, podemos subir o banco normalmente;



Localização dos control files:
```
SHOW PARAMETER control_files;
```
![[Pasted image 20260301085050.png]]

**Multiplexação dos Control Files** 

Como o control file é um arquivo crucial para a operação do banco de dados Oracle, é uma boa prática multiplexá-lo, ou seja, criar cópias operantes deste arquivo, que o banco de dados atualizará; 

O ideal é que estas cópias estejam em discos físicos diferentes, pois caso o disco venha a falhar e as cópias multiplexadas estejam no mesmo disco, a multiplexação é inefetiva;

O comportamento dos control files multiplexados é este: 
O banco de dados grava em todos os arquivos listados no parâmetro de inicialização CONTROL_FILES no arquivo de parâmetros de inicialização do banco de dados; 

O banco de dados lê apenas o primeiro arquivo listado no parâmetro CONTROL_FILES durante a operação do banco de dados; 

Se algum dos control files ficar indisponível durante a operação do banco de dados, a instância se tornará inoperável e deverá ser encerrada;

Nesse Contexto, irá ser criado um novo diretorio para salvar uma copia que sera multipexada do control file:

```
 mkdir /home/oracle/multiplex_control_file
```

Para multiplexação deve realizar uma alteração dentro do `spfile`:

```
ALTER SYSTEM SET CONTROL_FILES='/u02/oradata/ORCL/control01.ctl', '/u02/oradata/ORCL/control02.ctl', '/home/oracle/multiplex_control_file/control03.ctl' SCOPE=SPFILE;
```

Reinicialização do Banco para alterar o SPFILE:

```sql
SHUTDOWN IMMEDIATE;
```


Ao encerrar o banco de dados, realizar uma cópia do control file, para o novo diretorio:

```
cp /u02/oradata/ORCL/control01.ctl /home/oracle/multiplex_control_file/control03.ctl;

```

Conferindo:

```
ls -lh /home/oracle/multiplex_control_file;
```

![[Pasted image 20260301090714.png]]

Nesse sentido , já pode reinicialzar o banco:

```
STARTUP;
```


Verificando os parâmetros novamente:
```
SHOW PARAMETER control_files;
```
![[Pasted image 20260301095634.png]]


Realizando um backup do control_file de forma manual e resolvendo os problemas de conflito de versões.

-- Diferença da multiplexação:

Criação do diretório:

```
 mkdir /home/oracle/bkp_control_file;
```

```
ALTER DATABASE BACKUP CONTROLFILE TO '/home/oracle/bkp_control_file/control_file.bkp';
```
![[Pasted image 20260301101542.png]]

Após a Criação do backup a realização da criação de um tablaspace que atualizara o controlfile:

```
CREATE TABLESPACE TESTE_CONTROL_FILES;
```
![[Pasted image 20260301131609.png]]



Remoção acidental do control01 acreditando na eficiencia do backup manual

```
SHOW PARAMETER control_files;
```
![[Pasted image 20260301103143.png]]

```
cd /u02/oradata/ORCL;
rm -f control01.ctl;
```
![[Pasted image 20260301131902.png]]


Tentativa de criar uma nova tablaspace:
```
CREATE TABLESPACE TESTE_CONTROL_FILES2;
```
![[Pasted image 20260301132102.png]]

Já não é mais possivel realizar criações, devido não existir mais control_files;

Reiniciando a instância na tentativa de solucionar o problema;

![[Pasted image 20260301132243.png]]

Immediate falhou, utilizando o abort pra encerrar de forma forçada

![[Pasted image 20260301133017.png]]


Restaurando o control01 a partir da copia de backup realizada durante as alterações




```
cp /home/oracle/bkp_control_file/control_file.bkp /u02/oradata/ORCL/control01.ctl;
```
![[Pasted image 20260301133439.png]]


Inicializando a intancia novamente:
![[Pasted image 20260301134230.png]]

intancia subiu com erro de versão, reconheceu o controlfile, porém falhou, devido o backup realizado ser antigo e ter ocorrido operações por ser um arquivo multiplexado, para fehcar a instancia e copiar o aquivo atual para o mais recente, operação demanda cuidado, caso confuda poderá gerar diversas falhas de versão:


```
cp /u02/oradata/ORCL/control02.ctl /home/oracle/bkp_control_file/control_file.bkp



cp /home/oracle/bkp_control_file/control_file.bkp /u02/oradata/ORCL/control01.ctl
```

![[Pasted image 20260301135148.png]]

![[Pasted image 20260301135358.png]]


![[Pasted image 20260301135457.png]]


