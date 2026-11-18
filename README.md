# Territory Predictive Insights — Clima Local

Plataforma de monitoramento territorial (Global Solution 2º/2026 – POO, Turma de Fevereiro).
Recorte: **Clima local — chuva intensa e risco de alagamento em pontos vulneráveis.**

O sistema lê sensores simulados (chuva, nível do rio e nível do bueiro), grava cada medição
válida no banco H2, gera previsões com duas estratégias e emite alertas (ATENÇÃO, ALERTA,
CRÍTICO) quando a previsão atinge os limiares.

## Integrantes:
- Leonardo Luster Gomes – RM564448
- Ana Luiza Santos Bertão – RM563171
- Sofia Bill Franken – RM562767


## Como rodar

Pré-requisitos: **Java 17** e **Maven**.
```bash
mvn spring-boot:run
```
Rodar os testes:
```bash
mvn test
```
- O banco H2 fica em arquivo na pasta `data/` (criada ao rodar), então o histórico continua lá depois de reiniciar.
- As exportações CSV vão para a pasta `exportacoes/`.
- O código de acesso para alterar limiares está em `src/main/resources/application.properties` (`territorio.codigo-acesso`).

### Menu

1. Verificar sensores (lê e grava as medições)
2. Rodar previsões e alertas
3. Ver histórico: medições por sensor, alertas emitidos, exportar CSV
4. Gerenciar dados do território: região, população, limiares (com código de acesso)
5. Status completo
0. Sair

Dica: as previsões precisam de histórico. Use a opção 1 algumas vezes antes da opção 2.

## 1. Público beneficiado

**Quem sofre o problema:** os cerca de 45 mil moradores do Jardim Pantanal, bairro do
distrito do Jardim Helena, na Zona Leste de São Paulo. O bairro fica na várzea do Rio Tietê
e alaga com frequência quando chove forte. Nas enchentes de 2025 e 2026, parte das ruas
ficaram alagadas por vários dias seguidos. As casas mais expostas são as da beira do rio.

**Quem usa o sistema:** a equipe de uma organização comunitária do bairro (associação de
moradores ou grupo de apoio local) que acompanha a chuva, o nível do Tietê e a situação
das galerias e bueiros.

**Qual decisão o sistema apoia:** responder com antecedência à pergunta "vai alagar nos
próximos dias e o que fazemos hoje?". Com a previsão e o nível do alerta (ATENÇÃO, ALERTA
ou CRÍTICO), a equipe decide, por exemplo:
- avisar os moradores das ruas mais baixas para tirarem móveis, documentos e pertences do chão;
- organizar a saída de quem mora na margem do rio antes que a água suba;
- preparar um ponto de apoio ou abrigo;
- acionar a Defesa Civil e a subprefeitura com dados registrados (sensor, valor previsto,
limite atingido), e não só com relatos.

O sistema não decide sozinho: ele mostra o que os dados indicam e o porquê, e a decisão
fica com quem conhece o território.

## 2. Decisões da equipe

- **Recorte Clima local:** escolhemos chuva intensa e risco de alagamento porque é um problema recorrente atualmente, com impacto direto na vida dos moradores, e que pode ser medido por sensores simples.
- **Território: Jardim Pantanal (São Paulo/SP):** é um caso conhecido de alagamento
recorrente na beira do Tietê. O território é descrito por bairro e por ponto de
monitoramento, sem nenhum dado pessoal.
- **Três sensores:** pluviômetro (chuva em mm/h), nível do rio (m) e nível do bueiro
(% ocupado). Juntos eles cobrem as três causas do alagamento: a chuva que cai, o rio que
transborda e a drenagem que não dá conta.
- **Limiares de referência:** chuva 10/25/50 mm/h, rio 3/4/5 m e bueiro 60/80/95 %
(atenção/alerta/crítico). São valores iniciais, não calibrados para o bairro, e podem ser
alterados pelo menu com o código de acesso.
- **Spring Boot + Spring Data JPA com H2 em arquivo:** os repositories eliminam o SQL
escrito à mão (portanto não há concatenação de String) e o Spring gerencia as conexões.
O H2 em arquivo dispensa instalar um banco para a correção e mantém o histórico depois de
reiniciar o programa.
- **Duas estratégias rodando juntas:** a média móvel (últimas 5 leituras) e a comparação
com o período anterior (3 leituras contra as 3 anteriores) rodam sobre o mesmo histórico.
O serviço percorre uma lista de `EstrategiaPrevisao` e chama `prever()` de cada uma, sem
`if` de tipo nem `instanceof`. Por isso não foi preciso uma Factory para escolher a
estratégia.
- **Mínimo de 5 leituras:** com menos que isso o sistema não prevê e explica o motivo,
para não emitir alerta com base em quase nenhum dado.
- **Leituras inválidas não são gravadas:** a simulação inclui falhas do sensor e leituras
defeituosas (5% de chance cada) para mostrar as exceções funcionando. Essas leituras são
descartadas com uma mensagem e não entram no histórico.
- **Builder no relatório:** veja a seção 4.
- **Bônus: exportação CSV:** o histórico consultado de um sensor pode ser exportado para
`exportacoes/`, para ser aberto numa planilha ou compartilhado com a Defesa Civil.
- **Java 17:** versão LTS, a mais segura para o projeto compilar na máquina da correção.

## 3. Divisão de tarefas

| Integrante | Responsabilidade |
|---|---|
| Leonardo Luster Gomes RM564448  | Domínio e exceções |
| Ana Luiza Santos Bertão RM563171 | Persistência |
| Sofia Bill Franken RM562767 | Previsões, alertas e testes |

## 4. Padrão criacional: Builder (`RelatorioBuilder`)

O relatório de **status completo** junta partes de origens diferentes: dados do território
e limiares, pontos de monitoramento com o estado de cada sensor, última medição de cada
sensor e últimos alertas emitidos. Com o Builder, o menu monta o relatório passo a passo,
escolhendo quais seções entram e em que ordem:
```java
new RelatorioBuilder()
    .comTitulo(...)
    .comDataGeracao(...)
    .comDadosTerritorio(...)
    .comPontos(...)
    .comUltimasMedicoes(...)
    .comAlertasEmitidos(..., 5)
    .construir();
```
**Por que ali e não em outro lugar:** o relatório é o único objeto do sistema montado por
partes opcionais e em sequência. Um construtor com todos esses parâmetros ficaria grande e
difícil de ler. Já `Medicao` e `Alerta` são objetos simples, criados de uma vez e com
validação no próprio construtor, então um Builder neles só acrescentaria código. O
construtor de `Relatorio` é de pacote, então o resto do sistema só cria relatórios pelo
Builder e não repete a lógica de montagem.

## 5. Reflexão crítica

**Quando o sistema deve se calar?**
Com menos de 5 leituras no banco, o sistema não faz previsão e mostra o motivo (por
exemplo: "histórico insuficiente: 3 leitura(s) no banco, mínimo 5"). A comparação com o
período anterior precisa de 6 leituras (dois períodos de 3). Leituras com falha do sensor
ou com valor fora da faixa física são descartadas e não entram no histórico. Assim, um
sensor com defeito não gera um alerta falso.

**O que os dados não representam?**
Um sensor por ponto diz pouco sobre a rua ao lado; os valores são simulados com
Random e não seguem o comportamento real da chuva; os limiares são valores de
referência e não foram calibrados para este território.

**Por que o alerta apoia a decisão da pessoa, em vez de decidir por ela?**
o alerta mostra qual sensor, o valor previsto, o limite atingido e qual
estratégia gerou a previsão, para que a pessoa confira e decida (avisar moradores,
acionar a Defesa Civil etc.) com o conhecimento local que o sistema não tem.

**Uso de IA:**
A Inteligência Artificial foi utilizada como apoio durante o desenvolvimento do projeto, 
principalmente para esclarecer dúvidas sobre conceitos de Java, POO, persistência e testes, 
além de auxiliar na compreensão de erros encontrados durante a implementação. Os exemplos 
e sugestões gerados foram analisados e adaptados pela equipe, que revisou e compreendeu o 
código utilizado, mantendo a autoria e responsabilidade sobre a solução final. A IA foi 
utilizada também para nos ajudar na estrutura do README.
