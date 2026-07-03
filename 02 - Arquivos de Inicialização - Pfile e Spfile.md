


**SPFILE e PFILE: Como Gerenciar e Recuperar os Arquivos de Inicialização no Banco de dados Oracle**

  Durante o processo de `STARTUP` (Abertura e acesso ao banco de dados), dois arquivos de inicialização são fundamentais para que a instância se torne operacional: o SPFILE e o PFILE.

**SPFILE (Server Parameter File):**

- É um arquivo binário com capacidade leitura/escrita responsável por armazenar os parâmetros de inicialização da instância.
- Alterações feitas nos parâmetros enquanto a instância está em execução podem ser aplicadas dinamicamente e são persistentes, ou seja, serão mantidas após o encerramento e reinício da instância.
- Por ser um arquivo binário, o SPFILE não pode ser editado diretamente em um editor de texto; as modificações devem ser feitas através do seguinte comando em linguagem SQL:
```sql
ALTER SYSTEM SET parameter_name = value [SCOPE=MEMORY|SPFILE|BOTH];
```

**PFILE (Parameter File):**

- É um arquivo texto, com acesso somente leitura pelo Oracle Database durante a inicialização.
- O banco de dados apenas lê este arquivo para carregar seus parâmetros; ele não realiza escrita no PFILE.

Quando o comando `STARTUP` é executado **sem especificar explicitamente um PFILE**, o Oracle, por padrão, tenta encontrar e carregar o **SPFILE** para iniciar a instância. Caso o SPFILE não esteja disponível, o banco de dados inicializará utilizando o PFILE.

Neste laboratório pratico, o servidor de banco de dados é hospedado em uma maquina virtual com sistema operacional  baseado em Linux. Neste cenário por padrão os arquivos de inicialização podem ser localizados nos seguinte diretório:

```
ls -lh $ORACLE_HOME/dbs;
```
<img width="930" height="184" alt="image" src="https://github.com/user-attachments/assets/922d206e-ed81-40fb-9cad-f0d53b07a608" />


  Para aprofundar nos conceitos teóricos acima, pode se testar todas as hipóteses em ambiente real. Como todo fundamento de banco de dados, jamais deverá ser realizado testes em bases de produção. 

Antes de iniciar os testes por segurança é fundamental criar um backup  do PFILE para garantir uma restauração rápida caso ocorram falhas durante as alterações.

Neste cenário será criado  um diretório alternativo, fora do caminho padrão dos arquivos de inicialização, dentro da pasta `/home/oracle`:
```
mkdir /home/oracle/backup_pfile;
```

Criação de uma cópia do arquivo PFILE (Utilizando linguagem SQL):
```sql
CREATE PFILE='/home/oracle/backup_pfile/pfile_bkp.ora' FROM SPFILE; 
```
<img width="930" height="91" alt="image" src="https://github.com/user-attachments/assets/ee5b4894-a997-48d8-99d2-5b32c14120a2" />

Lembrando: Para criar uma cópia de segurança desse tipo de arquivo a instância não precisa estar necessariamente aberta. Mesmo com uma instancia ociosa é possivel realizar uma cópia binaria.


Visualizando o arquivo  no diretório criado:
```
ls -lh /home/oracle/backup_pfile;
```
<img width="930" height="74" alt="image" src="https://github.com/user-attachments/assets/582a4a55-3132-4e4d-a864-526b27029c3c" />


Caso o caminho não seja especificado, o Oracle salva o arquivo PFILE no diretório padrão, que normalmente é `$ORACLE_HOME/dbs`.

Após criar a cópia de segurança, será excluído os arquivos originais no diretório padrão e analisar o que ocorre durante esse processo.
```
 cd $ORACLE_HOME/dbs;
 ls -lh;
 rm -f initorcl.ora;
 rm -f spfileorcl.ora;
 ls -lh;
```
<img width="930" height="74" alt="image" src="https://github.com/user-attachments/assets/75b8b2a1-091f-4ea2-8164-e6be00f288fb" />

Durante a remoção dos arquivos de inicialização, a instancia estava em modo `open`(aberta).  Caso seja removido antes desse estado, não será possivel abrir o banco de dados.

Porém, apesar da remoção dos arquivos, a instância continuou rodando normalmente. Neste sentido, iremos tentar alterar o parâmetro `process` no spfile.
```
ALTER SYSTEM SET processes=500 SCOPE=SPFILE;
```
<img width="930" height="200" alt="image" src="https://github.com/user-attachments/assets/9c260e06-5258-4889-a0ee-f56e2bab1ebe" />

Retorna o erro ORA-01565: Arquivo de spfile não encontrado.

Por padrão, acredita-se que na tecnologia uma das melhores formas de resolver problemas básicos é reinicialização de um software/adware. Baseando nessa hipótese iremos reiniciar a instancia de Banco de Dados.
```
SHUTDOWN IMMEDIATE;
STARTUP;
```
<img width="930" height="145" alt="image" src="https://github.com/user-attachments/assets/d9333152-f0dd-4df8-8e61-54054040ebdc" />

Retorna o erro ORA-01078: Fala acessar os parâmetros do sistema.
O Alerta LRM-00109  tem se o seguinte retorno: Não foi possivel acessar o Pfile.

Nesse contexto, surge a seguinte duvida: O erro deveria ser no spfile? O spfile é o arquivo padrão para iniciar uma instancia Oracle.

Em resumo, O Oracle primeiro vai ate o spfile, caso não encontre ou depare com erro, o Pfile torna se o arquivo responsável pela inicialização. Em nosso cenário ambos foram removidos. Logo não existem parâmetros para o banco inicializar.


**Solução:** Neste modelo de desastre onde o DBA teve a precaução de realizar a copia de backup antes das alterações, essa cópia pode ser usada para tornar o ambiente operante novamente.

O comando em SQL pode ser realizado da seguinte maneira, passando o tipo arquivo + diretório onde foi salvo a cópia binaria:
```
STARTUP PFILE='/home/oracle/backup_pfile/pfile_bkp.ora'
```
<img width="930" height="174" alt="image" src="https://github.com/user-attachments/assets/3172f877-5775-4e8e-8a49-eac9cf1d7bfc" />


A instancia tornou aberta com sucesso. Logo pode ser testado alteração de um parâmetro de inicialização:
```
ALTER SYSTEM SET processes=500 SCOPE=SPFILE;
```
<img width="930" height="133" alt="image" src="https://github.com/user-attachments/assets/6b24b133-605e-4b75-b936-364190691d5a" />

Retorna o erro ORA-32001: Spfile não esta em uso.

Embora a instância tenha sido recuperada através do PFILE, o SPFILE permanece fora de operação.  Esse fundamento reforça o comportamento do Pfile, de não ser escrito pelo banco  de dados, apenas lido.

Para solucionar esse problema, existem duas alternativas:

1. Recuperar o SPFILE a partir dos dados em memória;
Contexto: Após abertura de uma instância, parâmetros de inicialização podem ser encontrados na memoria durante período online. 
    
2. Criar o SPFILE novamente utilizando a cópia do próprio PFILE em uso.
Contexto: O Spfile e Pfile podem ser criados utilizando um ao outro como base, a principal diferença entre ambos é que somente o Spfile pode ser alterado, e após isso o Pfile deve ser atualizado.

```
CREATE SPFILE FROM PFILE='/home/oracle/backup_pfile/pfile_bkp.ora'
```

Em nosso ambiente, recuperaremos o spfile, a partir de dados da memoria:
```
CREATE SPFILE FROM MEMORY;
```
<img width="930" height="110" alt="image" src="https://github.com/user-attachments/assets/4b4fae30-61cd-4ab0-93b2-7788fce94d8f" />



Conferindo o diretório padrão:
```
!ls -lh $ORACLE_HOME/dbs;
```
<img width="930" height="233" alt="image" src="https://github.com/user-attachments/assets/9947e4e6-9f20-48d4-828e-b371dcc48848" />

Ao listar o diretório padrão dos arquivos de inicialização, nota-se a presença do spfile, mas não é possivel ver o pfile padrão. Sure novamente duvidas: Como o banco está aberto utilizando Pfile? Se ele não esta listado no diretório. Isso ocorre pois o Pfile em uso foi passado via diretório de forma manual.


Iremos reiniciar a instancia apenas com o STARTUP simples, deixado por parte do Oracle escolher qual arquivo utilizar

```
SHUTDOWN IMMEDIATE;
STARTUP;
```

<img width="930" height="177" alt="image" src="https://github.com/user-attachments/assets/270324fb-acfa-4157-956e-66b414bac877" />



Novamente a tentativa de alterar  o parametro process

```
ALTER SYSTEM SET processes=500 SCOPE=SPFILE;
```
<img width="930" height="89" alt="image" src="https://github.com/user-attachments/assets/9585afd3-7356-4c88-a578-8a8b7a6a5164" />


Alteração realizada com sucesso! O SPFILE foi recuperado e está em uso. A partir dele, podemos gerar um PFILE para manter ambos os arquivos atualizados no diretório padrão.

```
CREATE PFILE FROM SPFILE;
```
<img width="930" height="98" alt="image" src="https://github.com/user-attachments/assets/ca709b5e-51a5-457b-8ede-5f8d634db8fe" />


Ao verificar o diretório padrão, ambos os arquivos estarão presentes:
```
!ls -lh $ORACLE_HOME/dbs;
```


**Conclusão:**  
  Ao realizar alterações nos parâmetros de inicialização do banco de dados, existe o risco de configurar incorretamente a instância, o que pode resultar em sua indisponibilidade. Por isso, é fundamental manter uma cópia de segurança atualizada do arquivo PFILE antes de qualquer modificação. Essa prática simples e preventiva garante uma via rápida e segura para a recuperação do ambiente, minimizando o tempo de inatividade e o impacto operacional.

Comandos Linux Utilizados:

ls -l: Lista os arquivos do diretorio; mkdir: Cria um novo diretorio; cd: acessa um diret quando permitido acesso.



Referências: 
DBAOCM 
https://mentoria.dbaocm.com/

Managing Initialization Parameters Using a Server Parameter File
https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/creating-and-configuring-an-oracledatabase.html#GUID-7302C60F-E96E-4202-AC81-25A6C93EEFA3

Specifying Initialization Parameters
https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/creating-and-configuring-an-oracledatabase.html#GUID-052F49CA-731A-4608-A2B9-2C801621D80F


Initialization Parameters
https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/initialization-parameters-2.html#GUIDFD266F6F-D047-4EBB-8D96-B51B1DCA2D61

