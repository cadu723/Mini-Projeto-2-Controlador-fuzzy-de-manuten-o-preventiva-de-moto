# Mini-Projeto 2 — Controlador fuzzy de manutenção preventiva de moto

**Disciplina:** Sistemas Baseados em Conhecimento (UFPB) — Prof. Daniel Faustino
**Caminho escolhido:** B (reescrita do domínio do MP1 com lógica fuzzy)
**Equipe:** Caio Eduardo · David Alves Noberto

## 1. Descrição do domínio

O projeto é um controlador fuzzy Mamdani, implementado em scikit-fuzzy, que indica o quão urgente é levar uma moto à oficina. A saída é uma nota de 0 a 100 calculada a partir de duas leituras: os quilômetros rodados desde a última troca de óleo (`km_oleo`) e a folga da corrente de transmissão (`folga_corrente`, em centímetros).

O domínio é o mesmo do Mini-Projeto 1 (diagnóstico de motos com regras IF–THEN em experta), agora com inferência fuzzy. Um mecânico não raciocina com limiares exatos: conceitos como "óleo vencido" e "corrente solta" não têm fronteira nítida. No MP1, uma moto com 1999 km de óleo era "óleo bom" e com 2000 km passava a "óleo vencido". O controlador fuzzy faz a urgência crescer de forma gradual.

## 2. Modelagem fuzzy

Cada variável tem três termos linguísticos, com funções trapezoidais nas pontas e triangulares no centro, sobrepostas nas transições para que a saída varie sem saltos.

| Variável | Universo | Termos | Pontos | Justificativa |
|---|---|---|---|---|
| `km_oleo` (entrada) | 0 a 4000 km | novo · uso_moderado · vencido | [0,0,1500,2000] · [1500,2500,3500] · [3000,3800,4000,4000] | Assume-se troca recomendada por volta de 3000 km. "Novo" vai até metade desse intervalo, "uso_moderado" fica entre os extremos e "vencido" começa em 3000 km e é pleno a partir de 3800 km. |
| `folga_corrente` (entrada) | 0 a 6 cm | justa · aceitavel · perigosa | [0,0,2,3] · [2,3,4] · [3,4.5,6,6] | Manuais costumam indicar folga entre 2 e 3,5 cm. "Justa" está na faixa boa, "aceitavel" no limite e "perigosa" solta demais. |
| `urgencia` (saída) | 0 a 100 | baixa · media · alta | [0,0,25,50] · [25,50,75] · [50,75,100,100] | Três níveis de ação, sobrepostos nas transições. |

Os valores de 3000 km e da faixa de folga são premissas da equipe, baseadas em recomendações usuais para motos de baixa cilindrada; o valor exato varia conforme o modelo e o tipo de óleo.

## 3. Base de regras

São 9 regras, uma para cada combinação de termos das duas entradas (matriz 3×3 completa), então não há lacuna no espaço de entradas. Critério: quanto pior o óleo e a corrente, maior a urgência.

| óleo \ folga | justa | aceitavel | perigosa |
|---|---|---|---|
| **novo** | R1: baixa | R2: baixa | R3: media |
| **uso_moderado** | R4: baixa | R5: media | R6: alta |
| **vencido** | R7: media | R8: alta | R9: alta |

Motor de inferência Mamdani do scikit-fuzzy: o E entre antecedentes usa o **mínimo**, a agregação das regras usa o **máximo** e a defuzzificação é pelo **centroide**. Como o centroide de um conjunto com largura nunca toca a borda do universo, a saída fica sempre entre cerca de 19 e 81.

```
km_oleo ─┐
         ├─► fuzzificação ─► 9 regras (min) ─► agregação (max) ─► centroide ─► urgência
folga ───┘
```

## 4. Como executar

**Google Colab (recomendado, sem instalar nada):**

1. Abra https://colab.research.google.com e vá em **Arquivo → Fazer upload de notebook**.
2. Envie o arquivo `entregavel2sbc.ipynb` deste repositório.
3. Clique em **Ambiente de execução → Executar tudo**. A primeira célula instala o `scikit-fuzzy`.

**Localmente (Python 3.9+):**

```bash
pip install scikit-fuzzy numpy scipy networkx jupyter
jupyter notebook entregavel2sbc.ipynb
```

Saída esperada:

```
Teste 1 (Ideal) - Urgência calculada: 19.44
Teste 2 (Fronteira) - Urgência calculada: 33.23
Teste 3 (Extremo) - Urgência calculada: 80.56
```

## 5. Casos de teste

| Caso | km_oleo | folga | Regras ativadas | Urgência | Interpretação |
|---|---|---|---|---|---|
| 1 — Ideal | 500 | 1.0 | R1 | 19,44 | Nada urgente. |
| 2 — Fronteira | 1800 | 2.5 | R1, R2, R4, R5 | 33,23 | Entre baixa e média. |
| 3 — Extremo | 3900 | 5.5 | R9 | 80,56 | Ir à oficina logo. |

No caso 2, 1800 km é "novo" com grau 0,4 e "uso_moderado" com 0,3, e 2,5 cm é "justa" e "aceitavel" com 0,5 cada. Quatro regras disparam juntas e o centroide combina todas, gerando uma urgência intermediária que uma regra nítida não conseguiria expressar.

## 6. Fuzzy × regras do MP1

No MP1, a regra era nítida: com 1999 km a moto tinha "óleo bom" e com 2000 km disparava "óleo vencido", levando a uma recomendação de troca. Uma diferença de 1 km mudava a decisão. No controlador fuzzy, com corrente justa, 1999 km e 2001 km dão urgência de 22,03 e 22,02: praticamente a mesma coisa, como na realidade.

| Aspecto | MP1 (regras nítidas, experta) | MP2 (fuzzy, scikit-fuzzy) |
|---|---|---|
| Limiar | Degrau rígido | Transição gradual |
| Saída | Rótulo/ação | Valor contínuo de 0 a 100 |
| Regras aplicáveis | Disparam uma por vez, em cadeia | Disparam juntas e são combinadas |
| Explicação | Trace direto das regras disparadas | Regras disparadas e seus graus |
| Escala | Cada limiar novo é mais uma regra | Regras crescem como kⁿ (k termos, n variáveis) |

Em complexidade, a versão nítida é mais simples de escrever, mas cada novo limiar exige mais uma regra. A versão fuzzy dá mais trabalho inicial (modelar termos e funções de pertinência), mas as 9 regras ficam legíveis por um especialista. Em expressividade, o fuzzy entrega uma saída suave e gradual, enquanto o MP1 devolve um rótulo, mas encadeia raciocínios em vários níveis com mais naturalidade.
