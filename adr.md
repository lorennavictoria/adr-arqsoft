# ADR-0001 01/02/2025 Separação da API de Manipulação de Dados como Microserviço

**Status:** Decidido 
**Created:** 2025-07-02  
**Tags:** microservices, data-processing, messaging 

---

## Contexto

A solução DataFlow é a aplicação central em nosso sistema, responsável por captar arquivos do usuário, aplicar transformações definidas por ele e entregar esses dados processados para consumo ou análise. Um dos seus principais componentes é a **API de Manipulação de Dados**, que executa pipelines complexos utilizando Apache Spark e Python.NET.

O sistema base foi desenvolvido utilizando .NET Core, que não oferece mais suporte ao Apache Spark. Quando surgiu a necessidade de implementar transformações de dados em larga escala— algo que, no contexto do projeto, só era viável com o Apache Spark — não foi possível integrar o serviço diretamente na aplicação. Como alternativa, optou-se por utilizar Python, uma linguagem compatível com o Spark, através do pacote do PySpark.

Para garantir a transferência de dados da API .NET para o Python sem perdas, foi adotado o Python.NET, que permite a integração entre as duas tecnologias. No entanto, essa solução apresenta desafios, já que o Python.NET possui suporte limitado e documentação escassa, dificultando a manutenção do sistema.

Além disso, esse acoplamento introduz dependências tecnológicas (como Spark e Python.NET) diretamente no DataFlow, aumentando o tempo de build, dificultando o desenvolvimento e testes locais, e limitando a flexibilidade de deploy.

Nosso objetivo com essa proposta é isolar o que é essencialmente um domínio separado — a manipulação de dados — e torná-lo escalável e independente.

## Decisão

Extrair a API de manipulação de dados do monólito da DataFlow e implementá-la como um **microserviço independente**, que se comunica com a DataFlow via **mensageria assíncrona**.

### Processo de tomada de decisão

O problema começou a se evidenciar à medida que o componente de manipulação de dados crescia em complexidade, especialmente com a integração de Apache Spark e Python.NET. Observamos que isso introduzia três impactos principais:

1. **Build e deploy mais lentos**: a inclusão do Spark, além da integração entre Python e .NET Core, tornaram o ciclo de desenvolvimento mais demorado e sujeito a falhas locais.
2. **Ambiente de desenvolvimento mais instável**: algumas versões do .NET Core/.NET 5+ podem ter problemas de compatibilidade.
3. **Impacto de performance e confiabilidade**: falhas em pipelines de dados podiam afetar todo o sistema.

Inicialmente, foi considerado **manter o design atual**, com a manipulação de dados embutida, e tentar modularizar internamente dentro da DataFlow. Isso simplificaria o deploy e evitaria novos serviços. No entanto, isso manteria o acoplamento tecnológico e dificultaria a escalabilidade independente.

Em seguida, foi explorada a possibilidade de **expor a API de manipulação de dados como um serviço REST síncrono**. Isso facilitaria a migração sem mudar tanto a lógica de integração atual. No entanto, foram identificados riscos importantes:
- Processamentos pesados poderiam manter conexões abertas por longos períodos;
- O modelo síncrono exigiria timeouts mais tolerantes.

Por fim, experimentamos um protótipo de **mensageria assíncrona**, utilizando mensagens JSON.
Simulamos o fluxo de uma transformação de dados como tarefa em fila e observamos:

- Redução do tempo de resposta da API principal, já que a tarefa é apenas enfileirada.
- Possibilidade de aplicar retries automáticos e separação de erros.
- Capacidade de paralelizar workers de forma independente da API principal.

Com esses resultados, optamos por seguir com a arquitetura baseada em eventos, e separar de forma clara os domínios: a DataFlow foca na captação de arquivos e na extração de dados, enquanto o novo microserviço se dedica à execução dos pipelines de dados.

Essa decisão também prepara o sistema para evoluções futuras — por exemplo, diferentes serviços de manipulação para tipos distintos de dados, sem impactar a API principal.

## Consequências

### Positivas

- **Isolamento de falhas**: falhas de processamento não afetam diretamente a API principal.
- **Escalabilidade independente**: a nova API poderá escalar conforme a carga de processamento.
- **Desacoplamento tecnológico**: a dependência do Apache Spark ficam restritas ao microserviço, simplificando a DataFlow. E não haverá mais a necessidade da Python.NET.
- **Facilidade de desenvolvimento**: permite pipelines de CI/CD e deploy separados.
- **Melhor controle de carga e retries**: com mensageria, é possível reprocessar mensagens com falha.

### Negativas

- **Aumento da complexidade arquitetural**: requer infraestrutura de mensageria, nesse caso, a infraestrutura do RabbitMQ.
- **Monitoramento mais complexo**: necessário rastrear o fluxo de mensagens e estado entre serviços.
- **Mais chamadas ao sistema de armazenamento**: O RabbitMQ encontra dificuldades para enviar arquivos muito grandes, nesse caso, a solução foi enviar o arquivo no DataFlow para o sistema de amazenamento e mandar por mensagem o path do arquivo. O serviço de manipulação de dados recebe o caminho e busca o arquivo no banco para processá-lo.
