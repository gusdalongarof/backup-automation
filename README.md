# Robô de Backup de Controles de Acesso (n8n Workflow)

Este é um fluxo de trabalho (workflow) do n8n projetado para automatizar o processo de login, backup de configurações, biometrias e cartões de múltiplos dispositivos de controle de acesso (provavelmente relógios de ponto ou terminais de acesso) e, em seguida, registrar o status da operação em uma planilha do Google Sheets e persistir os dados em um banco de dados Supabase.

O workflow também inclui um mecanismo de tratamento de erros que coleta os IPs que falharam e envia um alerta por e-mail.

## Visão Geral do Workflow

O fluxo principal do robô é o seguinte:

1.  **Gatilho Manual**: Inicia a execução.
2.  **Busca de Dados**: Carrega a lista de Controles de Acesso que precisam de backup de uma planilha do Google Sheets.
3.  **Preparação**: Formata a URL de login para cada dispositivo.
4.  **Loop Principal (`FAZER LOOP SOBRE TODOS OS CON`)**: Itera sobre cada dispositivo na lista.
    * **Login**: Realiza o login no dispositivo.
    * **Backup**:
        * `BACKUP CONFIGURAÇÕES`: Faz o download das configurações.
        * `SALVAR CONFIGURAÇÕES`: Salva o arquivo de configurações localmente.
        * `BACKUP BIOMETRIAS`: Faz o download dos dados de biometria.
        * `SALVAR BIOMETRIAS`: Salva o arquivo de biometrias localmente.
        * `BACKUP CARTÕES`: Faz o download dos dados dos cartões de acesso.
        * `SALVAR CARTÕES`: Salva o arquivo de cartões localmente.
    * **Registro de Sucesso**: Atualiza o status do dispositivo para "OK" na planilha do Google Sheets (`Update row in sheet`).
    * **Coleta de Dados**: Lê os arquivos de backup salvos localmente (`GET CARTÕES`, `GET BIOMETRIAS`, `Read/Write Files from Disk`).
    * **Extração de Texto**: Extrai o conteúdo de texto dos arquivos (`TXT CARTOES`, `TXT BIOMETRIAS`, `TXT CONFIGURACOES`).
    * **Persistência (Supabase)**: Insere uma nova linha no banco de dados Supabase com os dados do backup.
    * **Logout**: Executa o logoff no dispositivo.
    * **Tratamento de Erros**: Se alguma etapa falhar, a execução segue para o nó `Update row in sheet1` para registrar o erro na planilha do Google Sheets.

## Módulos Chave

| Módulo | Tipo | Função |
| :--- | :--- | :--- |
| `LISTA CONTROLES QUE PRECISAM DE BACKUP` | Google Sheets | Busca a lista de IPs e credenciais. |
| `FORMATAR URL` | Set | Cria a URL de login a partir dos dados da planilha. |
| `LOGIN CONTROLE DE ACESSO RH` | HTTP Request | Efetua o login no dispositivo de controle de acesso. |
| `BACKUP CONFIGURAÇÕES` | HTTP Request | Faz o download do arquivo de configurações (pgCode=9). |
| `BACKUP BIOMETRIAS` | HTTP Request | Faz o download dos dados de biometria (pgCode=17). |
| `BACKUP CARTÕES` | HTTP Request | Faz o download dos dados dos cartões (pgCode=10). |
| `SALVAR...` | Read/Write File | Salva os arquivos de backup na pasta `./backup/IP/`. |
| `Update row in sheet1` | Google Sheets | Registra falhas (status "ERRO") na planilha. |
| `Update row in sheet` | Google Sheets | Registra sucesso (status "OK") na planilha. |
| `Create a row` | Supabase | Persiste os dados de backup no banco de dados. |
| `FAZER LOGOFF` | HTTP Request | Encerra a sessão no dispositivo. |

## Tratamento e Alerta de Erros

O workflow possui um mecanismo de alerta centralizado:

1.  **Coleta de Falhas**: O nó `BUSCAR CONTROLES QUE FALHARAM` (Google Sheets) é acionado no final do loop principal e busca na planilha todos os IPs marcados com "ERRO".
2.  **Agregação**: O nó `Aggregate` coleta os dados de todos os IPs que falharam.
3.  **Geração de HTML**: O nó `Code in JavaScript` gera um corpo de e-mail formatado em HTML com uma tabela listando os IPs que apresentaram falha.
4.  **Alerta**: O nó `ENVIAR ALERTA` (Gmail) envia o e-mail para o destinatário `informatica9@hvidaesaude.org.br` com a lista de IPs problemáticos.

## Configuração

### Dependências Externas

Este workflow depende das seguintes integrações:

1.  **Google Sheets**: Requer credenciais OAuth2 para acessar e modificar a planilha.

2.  **Supabase**: Requer credenciais de API para inserção de dados.
   
