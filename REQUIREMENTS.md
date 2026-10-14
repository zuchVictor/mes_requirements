# 1.OBJETIVO

Fazer polling via HTTPS dos dados fornecidos pelos CLPs em cada máquina,
assimilá-los aos dados do client (ordem de serviço) e mantê-los em cache
para processamento de OEE por outro script. 

# 2.FLUXO

- Carregar configurações de yaml.
- Em loop:
    - Para cada CLP, fazer requisição GET HTTPS.
    - Validar conexão e integridade da resposta TLS de forma unidirecional do client com CA própria.
    - Validar schema dos dados recebidos.
    - Assimilar com dados do client por `clp_ip` e `machine_id`.
        - mapeamento é feito po `clp_ips.yaml` que contém cada ip de cada máquina 
    - Validar se informações batem com a OS de cada máquina
    - Atualizar cache.
- Repetir até receber sinal de parada.

# 3. CONFIGURAÇÃO

- CONFIG_PATH = `~/mes/config/`
    - FILES:
        - `clp_ips.yaml`
        - `paradas.yaml`
        - `produtos.yaml`
- CERT_PATH = `~/mes/certs/`
    - FILES:
        - `ca.pem` *client*
        - `server.dem` *CLP*
        - `server_key.dem` *CLP*
- TIMEOUT = 5
- POLL_INTERVAL = 1
- LOG_LEVEL = INFO

# 4.CONTRATO CLP

- Protocolo: HTTPS
- Método: GET
- Endpoint dos CLPs: /api/values
- Porta: 443
- Autenticação: TLS unidirecional (server-side only)
    - O CLP apresenta certificado ao client.
    - O client valida o certificado do CLP usando uma CA própria (ca.pem).
    - O client NÃO apresenta certificado ao CLP.
    - Sem autenticação de aplicação (token, basic auth) nesta versão.
    - Justificativa: limitação de heap do Pico W 2 inviabiliza mTLS
      (handshake mútuo consome ~25 KB adicionais de RAM).
- Verificação TLS:
    - Client: `requests.get(url, verify='<caminho>/ca.pem', timeout=5)`
    - CLP: servidor TLS com `server.der` + `server_key.der` carregados
      via `tls.SSLContext.load_cert_chain()`.
    - Certificado do CLP deve ter CN igual ao IP ou hostname de acesso.
    - Certificado assinado pela CA própria armazenada no client.- Timeout: 5s
- timestamp deve contar tempo com base no NTP
- Exemplo de request: GET https://192.168.1.100/api/values
- Exampl de response:

 ```
{
    dados: {
        ip: "198.168.0.10",
        machine_id: "Maq.30 80T",
        pcs_produzidas: 1000,
        pcs_refugo: 10,
        status_maquina: true,
        codigo_parada: null,
        timestamp: 5000,
    }
}
```

# 5.SCHEMA CLP

- Caso a informação não seja obrigatórioa, retorna valor null caso não esteja presente na leitura do CLP
- Caso não seja captadas o CLP deve retornar valor null
- Dados do CLP:

| Campo | Tipo | Observações | Obrigatório |
|:-:|:-:|:-|:-:|
| `ip` | `str` | info vem do CLP, validada com `clp_ips.yaml` | `True` |
| `machine_id` | `str` | info vem do CLP, validade com arquivo de programação da produção em CSV | `True` |
| `pcs_produzidas` | `int` | info vem do CLP, através de um rele contador de cada peça produzida, zera quando | `True` | 
| `pcs_refugo` | `int` | info vem do CLP, através de um rele contador de cada peça reprovada, zera quando | `True` |
| `status_maquina` | bool | info vem do CLP, True para quando pcs_produzidas teve incremento nos ultimos 5 minutos, False caso contrário indicando parada | `True` |
| `codigo_parada` | `int` | info vem do CLP, null se status_maquina=True, int para código de parada, indicado por paradas.yaml | `False` |
| `timestamp` | `int` | info vem do CLP, epoch time do CLP em segundos | `True` |

# 6.TRATAMENTO DE ERROS

- Tratamento de erros:
    - Timeout na requisição: tenta fazer a requisição mais 2 vezes, caso não tenha resposta do CLP então segue para a próxima leitura e realiza um log de erro de timeout
    - Falha na rede: Tentar Logar na rede novamente até sucesso ou atingir um limite de 10 segundos
    - CLP não envia JSON: tenta fazer a requisição mais 2 vezes, caso não tenha resposta do CLP então segue para a próxima leitura e realiza um log de erro de arquivo inválido
    - JSON corrompido/schema inválido: tenta fazer a requisição mais 2 vezes, caso não tenha resposta do CLP então segue para a próxima leitura e realiza um log de erro de dados inválidos
    HTTP 4xx: ir para próxmia leitura e realiza um log de erro de CLient 
    HTTP 5xx: tenta fazer a requisição mais 2 vezes, caso não tenha resposta do CLP então segue para a próxima leitura e realiza um log de erro de Server
    - Certificado TLS inválido: tenta fazer a requisição mais 2 vezes, caso não tenha resposta do CLP então segue para a próxima leitura e realiza um log de erro no TLS
- flag de registro: cada registro deve criar um log em caso de erro contendo o timestamp, o ip do CLP e o código do logger com a biblioteca de loggin do python, os códigos sendo:
    - INFO
    - DEBUG
    - WARNING
    - ERROR
    - CRITICAL

# 7.DADOS CLIENT

- Os dados virão do arquivo de programação da produção em CSV que contém:
 a progrmação de cada máquina para do dia, contendo qual a ordem de serviço, código do 
 produto, quantidade a ser produzida, e demais informações interessantes para  programação
 da produção
- Cada máquina pode ter várias ordens ao longo do dia, cada ordem deverá conter um timestamp de inicio e fim 
- Dados do client:

| Campo | Tipo | Observações |
|---------|:---------:|-----------|
| `machine_id` | `str` | info vem do CLP, serve para buscar os demais valores em maquinas.yaml |
| `id_ordem_servico` | `int` | valor vem do arquivo programação da produção e serve para rastreio do serviço |
| `codigo_produto` | `str` | valor vem da busca pela id_ordem_servico |
| `qtde` | `int` | valor vem da busca pela id_ordem_servico, vem como str e prcisa ser convertido em int após busca |

- Quando pcs_produzidas=qtde, indica fim de serviço
- O client deve ler os dados de um CLP e realizar o processo de obtenção dos dados do client 
 dentro de 1 segundo antes de guardar os dados em cache e seguir para a próxima leitura de 
 outro CLP, com timeout de 5 segundos
- A conversão de str em int em id_ordem_servico deve ser de um número positivo sem casas decimais
- Tabela exemplo:

| id_orden_servico | machine_id | codigo_produto | qtde |
|-|-|-|-|
| 08742 | "Maq.30 80T" | "PC_01" | 2000 |

- Reset dos contadores dos CLPs enviados por um segundo scrpit onde o clp é o Client para atualizar as informações do pedido quando o anterior é finalizado

# 8.CACHE

- Formato: SQLite para dados temporários 
- As chaves para encontrar os dados do CLP e Client será por id_ordem_servico como 
 chave primaria e machine_id como chave secundaria
- Todos os dados devem ser disponibilizados para que calculoOEE.py consiga identificar
 todas as instânceas obtidas pelo CLP + as informações chave do client para calcular o 
 OEE dos serviços em atividade/finalizados
- Retenção: deve conter dados semanais para garantia/audição, pós o intervalo os dados devem ser deletados
- Concorrência: O script deve por em fila os dados para que calculoOEE.py leia cada 1 
 e atualize a base de dados 
- estrutura exemplo:
```
{
    servicos: {
        id_ordem_servico: int,
        machine_id: str,
        pcs_produzidas: int,
        pcs_refugo: int,
        status_maquina: bool,
        codigo_parada: null | int,
        timestamp: int,
    }
}
```

# 9.OPERAÇÃO

- As execuções serão sequênciais
- O encerramento do processo finaliza quando
    - Todas as ordens de serviço do dia foram finalizadas em todas as máquinas
    - Todos os dados foram processados por calculoOEE.py
- Loggin: O Handler de loggin deve armazenar arquivos para cada semana de produção  
- Healthcheck CLP: Caso não haja atualização de um CLP por mais de 5 minutos, o script deve retornar um propt de ação para o responsável com uma mensagem de log caso o ultimo timestamp seje maior que o limite

# 10.REQUISITOS CLIENT

- Debian 13 (Trixie)
- Python 3.13
    - Dependências:
        - `requests`
        - `pyyaml`
        - `pydantic`
    - Módulos usados:
        - `json`
        - `time`
        - `datetime`
        - `sqlite3`
        - `pathlib`
        - `logging`
- Firmware CLP: MicroPython 
- Biblioteca no CLP: 
    - `tls` 
    - `socket`

# 11.CRITÉRIOS DE ACEITAÇÃO

- O scrip é considerado pronto quando:
    - É capaz de ler em loop de 1 segundo ao menos 2 CLPs
    - Capaz de conter todos os erros listados em na seção 6
    - Produz fila de cache sem falhas
