# Documentação do Scone

> Toda a documentação em `docs/` é **NORMATIVA**: ela deve permitir uma
> implementação independente sem ler o Rust. Qualquer divergência
> doc/código é um bug (identificar a fonte da verdade, corrigi-la).

Duas partes:

- **`general/`** — como funciona: os conceitos e as regras do protocolo,
  acessíveis sem ler os detalhes dos formatos;
- **`technical/`** — especificações detalhadas para desenvolvedores: formatos
  binários exatos, fórmulas de hash, limites, APIs e o comportamento de
  cada componente.

## `general/` — como funciona

| Documento | Conteúdo |
|---|---|
| [architecture.md](general/architecture.md) | Separação entre autoridade (blockchain) / disponibilidade (DHT) / armazenamento local; papéis das crates; fluxo de resolução DNS; evolutividade |
| [naming.md](general/naming.md) | Regras de nomenclatura: alfabeto LDH, TLDs, domínios e subdomínios, canonização, derivação de `DomainId` e representação hexadecimal |

## `technical/` — especificações para desenvolvedores

| Documento | Conteúdo |
|---|---|
| [protocol.md](technical/protocol.md) | Formato binário canônico do wire: varints LEB128, identificadores, registros DNS, blocos, mensagens P2P, limites, versionamento |
| [transactions.md](technical/transactions.md) | Formato v1 das transações assinadas: campos, ordem do wire (RegisterDomain carrega o nome, discriminantes arbitrários), derivação do owner, payload assinado `SCONE-TX-SIG-V1`, regras de validação |
| [blockchain.md](technical/blockchain.md) | Camada blockchain: `TxId`, árvore Merkle, `BlockHash`, gênese, estado canônico, validação de blocos, abstração de consenso |
| [keystore.md](technical/keystore.md) | Formato do arquivo de chaves `.sconekey` (Argon2id + XChaCha20-Poly1305), API do `scone-keystore`, comandos `identity` |
| [storage.md](technical/storage.md) | Persistência local redb: esquema de tabelas, deltas atômicos, política de memória, modelo de confiança, limites DoS |
| [relay.md](technical/relay.md) | Relay de rede (`scone-network`): protocolos libp2p, descoberta de pares (seeds, diretório, mDNS, UPnP), alcançabilidade (AutoNAT, circuit-relay, DCUtR), mempool, sync, DHT Kademlia, topologia local, métricas Prometheus, snapshot DNS, CLI completa, configuração `.env` |
| [dns.md](technical/dns.md) | Servidor DNS UDP + TCP (M6/M7b): snapshot DNS autoritativo, cache hot-path fragmentado em 16 vias (entradas compartilhadas, busca sem alocação, registros empacotados, compressão de nomes), codec RFC 1035, TCP RFC 7766 (orçamento 512 bytes UDP / 64 KiB TCP, bit TC), cache negativo DHT, fallback recursivo, limites |
| [cli.md](technical/cli.md) | Binário `scone`: log e verbosidade (stdout/stderr, níveis `-v`, `RUST_LOG`), configuração de ambiente via `.env` (precedência flag > `.env` > padrão) |
| [security-model.md](technical/security-model.md) | Modelo de segurança: seleção do comitê, anti-grinding de seed (análise: VRF desnecessária), equívoco e slashing |
| [audit-methodologie.md](technical/audit-methodologie.md) | Metodologia de auditoria de segurança: divergências, fonte da verdade, passagens de correção |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | RFC proof-of-service: heartbeats de operadores, delegação de operação, eleição do comitê entre assentos frescos |

## Convenções de referência cruzada

Os documentos internos e os comentários do código Rust referenciam estas
páginas pelo caminho absoluto a partir da raiz do repositório
(por ex. `/docs/en/technical/protocol.md`).
