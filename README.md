# 📊 Processo de Limpeza de Dados — PNS 2019
### Estudo pessoal sobre o projeto Capstone | Saúde de Idosos no Brasil

> **Objetivo da pesquisa:** Como o acesso a tratamentos de saúde entre idosos brasileiros varia em função de renda e região?
>
> **Dataset:** Pesquisa Nacional de Saúde (PNS) 2019 — IBGE  
> **Ferramenta:** Google Colab + Python  
> **ODS relacionado:** ODS 3 — Saúde e Bem-Estar

---

## 📁 Arquivos Utilizados

| Arquivo | Descrição |
|--------|-----------|
| `PNS_2019.txt` | Microdados brutos da pesquisa (formato de largura fixa) |
| `input_PNS_2019.txt` | Define as posições de cada variável no arquivo de dados |
| `dicionario_PNS_microdados_2019.xls` | Explica o significado de cada variável e seus códigos |
| `Chaves_PNS_2019.pdf` | Explica como identificar domicílios e pessoas no dataset |

---

## 🧠 Entendendo o Formato dos Dados

Os microdados do IBGE **não vêm em formato CSV** (com colunas separadas por vírgula). Eles vêm em formato de **largura fixa** — cada linha é uma string gigante onde cada posição tem um significado específico.

Por exemplo, uma linha pode ser assim:
```
11 0025 1 2 3...
```
E o arquivo `input_PNS_2019.txt` diz que:
- Posições 1-2 = Estado (UF)
- Posições 117-119 = Idade
- Posição 355 = Plano de saúde
- etc.

Por isso precisamos do arquivo de input para saber onde cada variável começa e termina.

---

## 📖 Entendimento das Variáveis (Data Understanding)

Antes de escrever qualquer código, consultamos o **dicionário de variáveis** para identificar quais colunas eram relevantes para a nossa pesquisa.

### Variáveis selecionadas:

| Código | Nome | Descrição | Posição no arquivo | Valores possíveis |
|--------|------|-----------|-------------------|-------------------|
| `V0001` | UF | Unidade da Federação (estado) | 1-2 | 11=RO, 12=AC, 13=AM... 35=SP, 43=RS |
| `C008` | IDADE | Idade do morador na data de referência | 117-119 | 0 a 130 (anos) |
| `I00102` | PLANO_SAUDE | Possui plano de saúde médico particular? | 355 | 1=Sim, 2=Não |
| `J012` | CONSULTAS_MEDICO | Quantas vezes consultou médico nos últimos 12 meses | 387-389 | 1 a 365 |
| `VDF004` | FAIXA_RENDA | Faixa de rendimento domiciliar per capita | 1535 | 1 a 7 |

### Decodificação da Faixa de Renda (VDF004):
| Código | Significado |
|--------|-------------|
| 1 | Até 1/4 do salário mínimo |
| 2 | Entre 1/4 e 1/2 salário mínimo |
| 3 | Entre 1/2 e 1 salário mínimo |
| 4 | Entre 1 e 2 salários mínimos |
| 5 | Entre 2 e 3 salários mínimos |
| 6 | Entre 3 e 5 salários mínimos |
| 7 | Mais de 5 salários mínimos |

### Por que as variáveis vêm como números e não como texto?
O IBGE codifica todas as respostas categóricas como números para padronizar o processamento de milhões de respostas de entrevistas. O dicionário funciona como um "tradutor" entre os códigos e seus significados reais. Isso também reduz o tamanho do arquivo final.

---

## 💻 Código Completo Comentado

### Etapa 1 — Conectar o Google Drive

```python
# O Google Colab não salva arquivos entre sessões
# Por isso conectamos ao Google Drive para acessar os arquivos permanentemente
from google.colab import drive
drive.mount('/content/drive')
# Após executar, o Colab pede autorização para acessar o Drive
# Quando funciona, aparece: "Mounted at /content/drive"
```

### Etapa 2 — Verificar se os arquivos estão acessíveis

```python
import os

# Caminho da pasta onde estão os arquivos no Drive
caminho = '/content/drive/MyDrive/trabalhodoluciano'

# Lista todos os arquivos na pasta
os.listdir(caminho)
# Resultado esperado:
# ['dicionario_PNS_microdados_2019.xls', 'input_PNS_2019.txt', 'PNS_2019.txt']
```

### Etapa 3 — Ler o arquivo de input para descobrir as posições das variáveis

```python
# O arquivo input_PNS_2019.txt define onde cada variável começa no arquivo de dados
# Cada linha começa com @ seguido da posição, nome da variável e tamanho

colspecs = []  # Lista com (início, fim) de cada variável
names = []     # Lista com os nomes das variáveis

with open('/content/drive/MyDrive/trabalhodoluciano/input_PNS_2019.txt', 'r', encoding='latin-1') as f:
    # encoding='latin-1' é necessário porque o arquivo tem caracteres especiais do português
    for line in f:
        line = line.strip()
        if line.startswith('@'):  # Linhas que definem variáveis começam com @
            parts = line.split()
            start = int(parts[0].replace('@', '')) - 1  # -1 porque Python começa do 0
            name = parts[1]
            length = int(''.join(filter(str.isdigit, parts[2])))
            colspecs.append((start, start + length))
            names.append(name)

print(f"Total de variáveis encontradas: {len(names)}")
# Resultado: Total de variáveis encontradas: 1088
```

> ⚠️ **Por que `encoding='latin-1'`?** O arquivo foi criado com codificação antiga (Windows-1252/Latin-1) que suporta caracteres como ã, ç, é. Se não especificarmos isso, o Python não consegue ler e dá `UnicodeDecodeError`.

### Etapa 4 — Buscar as posições exatas das variáveis que precisamos

```python
# Em vez de carregar todas as 1088 variáveis (que causaria erro de memória),
# buscamos as posições exatas só das variáveis que precisamos

with open('/content/drive/MyDrive/trabalhodoluciano/input_PNS_2019.txt', 'r', encoding='latin-1') as f:
    for line in f:
        if 'C008' in line or 'I00102' in line or 'VDF004' in line or 'J012' in line:
            print(line.strip())

# Resultado:
# @00117	C008	3.       → Idade começa na posição 117, tamanho 3
# @00355	I00102	$1.    → Plano de saúde começa na posição 355, tamanho 1
# @00387	J012	3.       → Consultas médico começa na posição 387, tamanho 3
# @01535	VDF004	$1.    → Faixa de renda começa na posição 1535, tamanho 1
```

### Etapa 5 — Carregar apenas as colunas necessárias

```python
import pandas as pd

# Definindo apenas as 5 colunas que precisamos
# Formato: (posição_inicial, posição_final) — Python começa do índice 0
# Por isso subtraímos 1 da posição do dicionário
colspecs = [
    (0, 2),       # UF: posição 1-2 no dicionário → (0,2) no Python
    (116, 119),   # C008 (Idade): posição 117-119 → (116,119) no Python
    (354, 355),   # I00102 (Plano de saúde): posição 355 → (354,355)
    (386, 389),   # J012 (Consultas médico): posição 387-389 → (386,389)
    (1534, 1535), # VDF004 (Faixa de renda): posição 1535 → (1534,1535)
]

names = ['UF', 'IDADE', 'PLANO_SAUDE', 'CONSULTAS_MEDICO', 'FAIXA_RENDA']

# read_fwf = read fixed-width file (lê arquivo de largura fixa)
df = pd.read_fwf(
    '/content/drive/MyDrive/trabalhodoluciano/PNS_2019.txt',
    colspecs=colspecs,
    names=names,
    encoding='latin-1'
)

print(f"Linhas: {df.shape[0]}")   # shape[0] = número de linhas
print(f"Colunas: {df.shape[1]}")  # shape[1] = número de colunas
# Resultado: Linhas: 293726, Colunas: 5
```

> ⚠️ **Por que não carregamos todas as 1088 colunas?**  
> Na primeira tentativa, tentamos carregar todas as colunas e o Colab deu erro:  
> `"A sessão falhou depois de usar toda a RAM disponível."`  
> O Colab gratuito tem limite de memória RAM (~12GB). Com 293.726 linhas × 1.088 colunas, o arquivo ficou grande demais. A solução foi carregar só as 5 colunas que precisávamos.

### Etapa 6 — Verificar se as posições estão corretas

```python
# Antes de filtrar, verificamos se os dados fazem sentido
print("Idades únicas (primeiros 20 valores):")
print(sorted(df['IDADE'].dropna().unique())[:20])
# Se aparecer valores impossíveis (ex: 101, 102...) a posição está errada

# Na primeira tentativa, a posição da idade estava errada (usamos 56-59 em vez de 116-119)
# Os valores apareceram como 101, 102, 103... ao invés de 0, 1, 2, 3...
# Corrigimos buscando a posição certa no arquivo de input
```

### Etapa 7 — Filtrar apenas os idosos

```python
# Nossa pesquisa é sobre idosos (60 anos ou mais)
# Sem esse filtro, analisaríamos 293.726 pessoas de todas as idades
# o que incluiria bebês, crianças e adultos que não são relevantes para nossa pergunta

idosos = df[df['IDADE'] >= 60].copy()
# .copy() cria uma cópia independente para evitar avisos do pandas

print(f"Total de idosos: {idosos.shape[0]}")
# Resultado: Total de idosos: 43554
```

### Etapa 8 — Remover valores ausentes (NaN)

```python
# NaN = Not a Number = célula vazia
# Ocorre quando o entrevistado não respondeu uma pergunta
# Não podemos analisar dados que não existem, então removemos essas linhas

idosos_limpo = idosos.dropna(subset=['PLANO_SAUDE', 'CONSULTAS_MEDICO', 'FAIXA_RENDA'])
# subset=['...'] = verificar valores ausentes apenas nessas colunas específicas

print(f"Idosos antes da limpeza: {idosos.shape[0]}")        # 43554
print(f"Idosos após limpeza: {idosos_limpo.shape[0]}")      # 36999
print(f"Removidos: {idosos.shape[0] - idosos_limpo.shape[0]}") # 6555
```

### Etapa 9 — Criar a coluna de Região

```python
# A variável UF tem os códigos de cada estado (ex: 35=SP, 33=RJ, 43=RS)
# Para nossa análise, precisamos agrupar os estados em regiões

def regiao(uf):
    if uf in [11,12,13,14,15,16,17]:
        return 'Norte'
    elif uf in [21,22,23,24,25,26,27,28,29]:
        return 'Nordeste'
    elif uf in [31,32,33,35]:
        return 'Sudeste'
    elif uf in [41,42,43]:
        return 'Sul'
    elif uf in [50,51,52,53]:
        return 'Centro-Oeste'

idosos_limpo = idosos_limpo.copy()  # Evita o SettingWithCopyWarning
idosos_limpo['REGIAO'] = idosos_limpo['UF'].apply(regiao)
# .apply() aplica a função regiao() para cada valor da coluna UF

print(idosos_limpo['REGIAO'].value_counts())
# Resultado:
# Nordeste        12497
# Sudeste          9472
# Norte            5771
# Sul              5374
# Centro-Oeste     3885
```

### Etapa 10 — Decodificar as variáveis categóricas

```python
# Traduzindo plano de saúde: 1=Sim, 2=Não → texto legível
idosos_limpo['PLANO_SAUDE'] = idosos_limpo['PLANO_SAUDE'].map({
    1.0: 'Com plano',
    2.0: 'Sem plano'
})
# .map() substitui cada valor pelo seu correspondente no dicionário

# Traduzindo faixa de renda: números → salários mínimos
renda_map = {
    1.0: 'Até 1/4 SM',
    2.0: '1/4 a 1/2 SM',
    3.0: '1/2 a 1 SM',
    4.0: '1 a 2 SM',
    5.0: '2 a 3 SM',
    6.0: '3 a 5 SM',
    7.0: 'Mais de 5 SM'
}
idosos_limpo['FAIXA_RENDA'] = idosos_limpo['FAIXA_RENDA'].map(renda_map)

print("Plano de saúde:")
print(idosos_limpo['PLANO_SAUDE'].value_counts())
# Sem plano    26670
# Com plano    10329

print("\nFaixa de renda:")
print(idosos_limpo['FAIXA_RENDA'].value_counts())
```

---

## 🚨 Perrengues que Aconteceram

### Perrengue 1 — UnicodeDecodeError
**O que aconteceu:** Ao tentar ler o arquivo `input_PNS_2019.txt` sem especificar encoding, o Python deu erro:
```
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe7
```
**Por que aconteceu:** O arquivo foi criado com codificação `latin-1` (padrão antigo do Windows), não `utf-8` (padrão moderno).  
**Como resolvemos:** Adicionamos `encoding='latin-1'` na abertura do arquivo.

---

### Perrengue 2 — Sessão falhou por falta de RAM
**O que aconteceu:** Ao tentar carregar todas as 1088 colunas do dataset, o Colab mostrou:
```
A sessão falhou depois de usar toda a RAM disponível.
```
**Por que aconteceu:** 293.726 linhas × 1.088 colunas = arquivo enorme demais para a memória do Colab gratuito.  
**Como resolvemos:** Carregamos apenas as 5 colunas necessárias usando `colspecs` no `read_fwf()`.

---

### Perrengue 3 — Posição da variável IDADE estava errada
**O que aconteceu:** A coluna IDADE mostrava valores como 101, 102, 103 ao invés de idades reais.  
**Por que aconteceu:** Usamos a posição 56-59 quando a posição correta era 117-119.  
**Como resolvemos:** Buscamos a posição exata no arquivo `input_PNS_2019.txt` filtrando pela variável `C008`.

---

### Perrengue 4 — SettingWithCopyWarning
**O que aconteceu:** Ao criar a coluna REGIAO, o pandas mostrou um aviso:
```
SettingWithCopyWarning: A value is trying to be set on a copy of a slice from a DataFrame.
```
**Por que aconteceu:** O dataframe `idosos_limpo` era uma "fatia" do dataframe original, não uma cópia independente. Modificar uma fatia pode ter comportamentos inesperados.  
**Como resolvemos:** Adicionamos `.copy()` antes de criar a nova coluna, garantindo que trabalhávamos com uma cópia independente.

---

## 📊 Conclusões dos Dados

Após todo o processo de limpeza, chegamos a um dataset com **36.999 idosos brasileiros** com as seguintes características:

### Distribuição por Plano de Saúde
| Situação | Quantidade | Percentual |
|----------|-----------|------------|
| Sem plano de saúde | 26.670 | ~72% |
| Com plano de saúde | 10.329 | ~28% |

> **Conclusão:** A grande maioria dos idosos brasileiros (72%) não possui plano de saúde, dependendo exclusivamente do SUS para acesso a serviços de saúde.

### Distribuição por Região
| Região | Quantidade |
|--------|-----------|
| Nordeste | 12.497 |
| Sudeste | 9.472 |
| Norte | 5.771 |
| Sul | 5.374 |
| Centro-Oeste | 3.885 |

> **Conclusão:** O Nordeste concentra o maior número de idosos na amostra, seguido do Sudeste. Isso reflete tanto a distribuição populacional quanto a abrangência da pesquisa.

### Distribuição por Faixa de Renda
| Faixa | Quantidade |
|-------|-----------|
| 1/2 a 1 SM | 12.585 |
| 1 a 2 SM | 10.758 |
| 2 a 3 SM | 3.708 |
| 1/4 a 1/2 SM | 3.341 |
| 3 a 5 SM | 2.885 |
| Mais de 5 SM | 2.850 |
| Até 1/4 SM | 872 |

> **Conclusão:** A maioria dos idosos (mais de 60%) vive com até 2 salários mínimos per capita, o que reforça a hipótese de que o acesso a planos de saúde é limitado pela renda.

### Média de Consultas Médicas
- **Média geral:** 4,55 consultas nos últimos 12 meses
- **Máximo:** 365 consultas (provavelmente tratamento contínuo)
- **Mínimo:** 1 consulta

> **Observação importante:** A variável de consultas médicas (J012) só foi respondida por quem foi ao médico ao menos uma vez. Os 6.555 registros removidos incluem pessoas que podem não ter ido ao médico nenhuma vez — o que por si só já é um indicador de falta de acesso.

---

# 🎤 Roteiro de Apresentação — Data Storytelling
## "Acesso à Saúde entre Idosos Brasileiros: Uma Análise da Desigualdade por Renda e Região"
### Jessica de Toledo Hernandez | CCD210 | 2º Semestre 2026

---

## 🟣 PARTE 1 — O GANCHO
*Ainda sem abrir o dashboard. Fala direto para o professor.*

**O que falar:**

> "Eu quero que vocês imaginem dois idosos brasileiros.
>
> Os dois têm 70 anos. Os dois precisam de acompanhamento médico regular.
>
> O primeiro mora no interior do Maranhão. Aposentado, recebe menos de um salário mínimo por mês. Não tem plano de saúde — nunca teve. Quando precisa de consulta, pega ônibus por horas até o posto de saúde mais próximo.
>
> O segundo mora em Curitiba. Aposentado também, mas com uma renda maior. Tem plano de saúde há anos. Liga pro consultório e marca consulta para a semana seguinte.
>
> A diferença entre esses dois idosos não é sorte. É estrutura. É renda. É região.
>
> E os dados provam isso."

*[abre o dashboard.]*

---

## 🟣 PARTE 2 — CONTEXTO E DADOS
*Mostra o dashboard, aponta para os cards do topo.*

**O que falar:**

> "Para entender essa desigualdade, eu analisei a Pesquisa Nacional de Saúde de 2019, do IBGE — uma pesquisa pública com quase 300 mil brasileiros entrevistados.
>
> Filtrei apenas os idosos — pessoas com 60 anos ou mais — e cheguei a quase 37 mil registros válidos para análise.
>
> Logo de cara, os números já contam uma história preocupante."

*[Aponta para o card de 72%]*

> "72% dos idosos brasileiros não têm plano de saúde. 
>
> Pensa nisso: de cada 10 idosos no Brasil, 7 dependem exclusivamente do SUS para qualquer tipo de atendimento médico."

---

## 🟣 PARTE 3 — A HISTÓRIA DA RENDA
*Aponta para o gráfico de barras de renda.*

**O que falar:**

> "Esse gráfico aqui mostra o percentual de idosos SEM plano de saúde por faixa de renda.
>
> Olha a progressão."

*[Vai apontando barra por barra, de cima para baixo]*

> "Na faixa mais baixa — menos de um quarto do salário mínimo — 92% dos idosos não têm plano.
>
> Na faixa seguinte — de um quarto a metade do salário mínimo — são 95%. O maior percentual de todos.
>
> Conforme a renda vai subindo, o percentual vai caindo.
>
> E olha o extremo oposto: idosos que ganham mais de 5 salários mínimos? Apenas 12% estão sem plano.
>
> Em outras palavras: ter ou não ter plano de saúde no Brasil não é uma escolha. É uma consequência direta de quanto dinheiro você tem."

---

## 🟣 PARTE 4 — A HISTÓRIA DA REGIÃO
*Aponta para o gráfico de pizza de região.*

**O que falar:**

> "Mas a renda não é o único fator. A região onde você mora também determina seu acesso à saúde.
>
> Esse gráfico mostra a distribuição de idosos sem plano por região do Brasil."

*[Aponta para as fatias]*

> "Norte e Nordeste concentram os maiores percentuais — mais de 80% dos idosos nessas regiões não têm plano de saúde.
>
> No Sudeste esse número cai para cerca de 60%. Ainda alto, mas significativamente menor."

*[Usa o filtro interativo — clica em Nordeste]*

> "Vamos aplicar um filtro aqui. Olha o que acontece quando a gente isola apenas o Nordeste."

*[Mostra os números mudando]*

> "Os cards mudam. A média de consultas cai. O percentual sem plano sobe ainda mais.
>
> Agora vamos comparar com o Sul."

*[Troca o filtro para Sul]*

> "Sul. Olha a diferença. A média de consultas é maior. O percentual sem plano é menor.
>
> E se a gente combinar região com renda..."

*[Seleciona Sul + Mais de 5 SM]*

> "Sul, renda alta: acesso privilegiado à saúde.
>
> Agora..."

*[Seleciona Nordeste + Até 1/4 SM]*

> "Nordeste, renda baixa: praticamente sem acesso privado. 100% dependente do SUS.
>
> Esse é o Brasil que os dados mostram."

---

## 🟣 PARTE 5 — O GRÁFICO DE CONSULTAS 
*Aponta para o gráfico de barras de consultas por região.*

**O que falar:**

> "E como isso aparece no número de consultas médicas?
>
> Em todas as regiões, a barra roxa — idosos com plano — é sempre maior que a barra verde — idosos sem plano.
>
> Isso significa que ter plano de saúde impacta diretamente a frequência com que o idoso consegue ver um médico.
>
> E isso não é detalhe. Consultas regulares significam diagnóstico precoce. Significam tratamento contínuo de doenças crônicas como hipertensão e diabetes. Significam qualidade de vida."

---

## 🟣 PARTE 6 — CONCLUSÃO E IMPACTO 
*Remove todos os filtros. Mostra o dashboard completo.*

**O que falar:**

> "Então o que os dados nos dizem?
>
> Que no Brasil, o acesso à saúde na terceira idade é fortemente determinado por dois fatores: quanto você ganha e onde você mora.
>
> Lembram dos dois idosos que eu apresentei no começo? Os dados confirmam a história deles.
>
> O idoso do Maranhão, de baixa renda, no Nordeste — está no grupo com menor acesso. Mais de 80% dos idosos na situação dele não têm plano de saúde.
>
> O idoso de Curitiba, de renda maior, no Sul — está no grupo com maior acesso.
>
> Essa desigualdade é real. E ela tem consequências na saúde, na qualidade de vida e na longevidade dessas pessoas.
>
> Esse estudo está alinhado com o ODS 3 da ONU — Saúde e Bem-Estar para todos. Porque os dados mostram que ainda estamos longe de garantir esse direito de forma igual para todos os idosos brasileiros.
>
> Obrigada."

---






*Documento criado para estudo pessoal do processo de Data Understanding e Data Preparation com Python e Google Colab.*
