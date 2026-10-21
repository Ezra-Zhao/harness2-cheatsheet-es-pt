# 🔁 Guía rápida: Harness² — el agente que reescribe sus propias instrucciones (ES · PT)

[🇪🇸 Español](#-español) · [🇵🇹 Português](#-português) · [🇬🇧 English summary](#-english-summary)

Una sola página para entender **Harness²** (google-research): el framework que mejora el sistema que rodea a un modelo —sus prompts, su memoria, sus herramientas— sin tocar los pesos del modelo. El agente propone cambios a sus propias instrucciones, los prueba sobre la tarea real y se queda con lo que funciona.

**Autor:** Guangyi Zhao · Octubre 2026

Proyecto original: https://github.com/google-research/harness2 (14.369 estrellas y 1.324 forks, consultado el 20 de octubre de 2026) · Paper: arXiv:2610.14224

> **Nota de autoría:** esta guía es una síntesis original, redactada por el autor a partir de la documentación pública del proyecto y de su propio uso. No es una traducción ni una copia de la documentación oficial. El repositorio vive en la organización google-research, pero su propio README aclara que no es un producto de Google con soporte oficial. Para el detalle completo, ve al repositorio enlazado.

---


Apoyo y más: [GitHub Sponsors](https://github.com/sponsors/Ezra-Zhao) · [Gumroad — plantillas y cuadernos que uso](https://ezraflow28.gumroad.com) · [Ko-fi](https://ko-fi.com/ezra777zhao)

## 🇪🇸 Español

### Qué es un harness y qué hace Harness²

El **harness** es todo lo que rodea al modelo para que actúe como agente: los prompts, las herramientas, la memoria, las habilidades, la orquestación. Mejorar un agente casi nunca exige un modelo más grande; exige un harness mejor. Harness² automatiza esa mejora en tareas que el diseñador del harness nunca vio, con los pesos del modelo congelados.

Lo hace con dos niveles de recursión: dentro de cada tarea, un ciclo **Propose–Probe–Compose** refina el harness de esa tarea; entre tareas, un **harness de mejora persistente** guarda lo adoptado y las lecciones con evidencia, y guía el refinamiento siguiente. Lo aprendido en una tarea no se pierde: pasa a ser la biblioteca de experiencia de la siguiente.

### Los 8 componentes que puede reescribir

| Componente | Rol | Qué controla |
|---|---|---|
| **System prompt** | Construir el contexto | Las instrucciones base del agente en cada sesión |
| **Guardrails** | Construir el contexto | Los límites y reglas de conducta dentro del bucle |
| **Memoria** | Construir el contexto | Qué se conserva entre pasos y entre tareas |
| **Skills** | Construir el contexto | Capacidades empaquetadas que el agente puede invocar |
| **Tool guidance** | Interfaz de herramientas | Cómo se describen las herramientas al modelo y cómo se le enseña a usarlas |
| **Scripts** | Interfaz de herramientas | El código de apoyo que prepara y procesa el trabajo |
| **Subagents** | Delegación | Cómo se reparte el trabajo entre agentes delegados |
| **Plugins** | Middleware | El comportamiento transversal dentro del bucle de herramientas |

### El ciclo Propose–Probe–Compose

| Paso | Quién actúa | Qué hace |
|---|---|---|
| **Propose** | El editor de harness | Propone cambios sobre los 8 componentes: de dos tipos, **invariantes** (valen para cualquier tarea) y **de dominio** (específicos de esta clase de tarea) |
| **Probe** | Los ejecutores candidatos | Cada propuesta se ejecuta sobre **la misma tarea**; no hace falta disponer de la respuesta correcta ni de un verificador con la solución |
| **Compose** | El compositor | Compara las ejecuciones entre sí —el **contraste de comportamiento**— y arma el harness refinado con lo que demostró funcionar |

El presupuesto de recursión es `--k`: con `--k 1` hay un solo paso Propose–Probe–Compose. Hay dos maneras de escalarlo:

| Modo | Cómo escala | En una frase |
|---|---|---|
| `--mode parallel` | Prueba K candidatos generales y K de dominio, y produce K composiciones desde la evidencia compartida | Explora más ancho |
| `--mode sequential` | Encadena K pasos Propose–Probe–Compose llevando el harness refinado y la evidencia acumulada | Afina más hondo |

### Ejecutores y benchmarks

El ejecutor por defecto es **OpenCode**; con `--substrate cx` se usa **Codex**. Los resultados publicados corresponden a un solo paso de Harness² sobre los harness base (modelos GLM-5 y Gemini 3.5 Flash), según el paper y el README del proyecto:

| Benchmark | Qué mide | Instalación | Mejora publicada |
|---|---|---|---|
| **JobBench** | Trabajo profesional | `harness2-setup jb` | +4,5 puntos |
| **LAB** | Trabajo legal | `harness2-setup lab` | +11,8 puntos porcentuales en la tasa de aprobación parcial |
| **WorkBuddy-Bench** | Oficina, web y código | `harness2-setup wb` | Hasta +11,5 puntos |

### Empieza por la demo local (sin gastar en API)

La demo corre ambos modos de recursión con editores deterministas y ejecuciones Python reales, **sin credenciales de modelo**:

```bash
# Linux · Python 3.12 · uv
uv venv --python 3.12 .venv
source .venv/bin/activate
uv pip install -e '.[render]'
harness2-demo --output runs/demo --k 2
```

Los benchmarks con modelo sí cuestan: piden Node.js 20+, Google Cloud CLI y un proyecto de Vertex AI en el `.env`; LAB además necesita Pandoc y Poppler, y WorkBuddy-Bench, Docker. Empieza con `--max-tasks 1` y usa `--dry-run` para ver qué tareas se elegirían sin llamar al modelo. Los resultados quedan en `runs/`, y repetir el mismo comando reanuda donde quedó.

### Harness² frente a ECC, en una línea

ECC es un paquete de optimizaciones de harness reunidas a mano por personas: se instala tal cual y no cambia entre tareas (estático). Harness² genera, prueba y compone los cambios automáticamente en cada tarea nueva, y acumula lo aprendido (dinámico).

### ⚠️ Errores típicos

- **Juzgar con otra configuración.** `harness2-judge` y `harness2-report` deben usar el mismo benchmark, ejecutor, subconjunto, `--k`, `--mode` y semilla que la corrida de refinamiento; si no, los números no son comparables.
- **Evaluar con la red abierta.** El ejecutor tiene acceso web sin restricciones y Codex corre con la sandbox desactivada: las rúbricas públicas de los benchmarks pueden estar en línea. Configura las restricciones de red antes de evaluar, o el agente puede «encontrar» las respuestas en vez de resolver la tarea.
- **Quemar presupuesto sin ver la demo.** Sin haber visto el ciclo funcionar en local con la demo gratuita, es fácil gastar llamadas a modelos en una selección de tareas equivocada.

---

## 🇵🇹 Português

### O que é um harness e o que o Harness² faz

O **harness** é tudo o que envolve o modelo para que ele aja como agente: os prompts, as ferramentas, a memória, as habilidades, a orquestração. Melhorar um agente quase nunca exige um modelo maior; exige um harness melhor. O Harness² automatiza essa melhoria em tarefas que o projetista do harness nunca viu, com os pesos do modelo congelados.

Ele faz isso com dois níveis de recursão: dentro de cada tarefa, um ciclo **Propose–Probe–Compose** refina o harness daquela tarefa; entre tarefas, um **harness de melhoria persistente** guarda o que foi adotado e as lições com evidência, e guia o refinamento seguinte. O que se aprende numa tarefa não se perde: vira a biblioteca de experiência da próxima.

### Os 8 componentes que ele pode reescrever

| Componente | Papel | O que controla |
|---|---|---|
| **System prompt** | Construir o contexto | As instruções-base do agente em cada sessão |
| **Guardrails** | Construir o contexto | Os limites e as regras de conduta dentro do loop |
| **Memória** | Construir o contexto | O que se conserva entre passos e entre tarefas |
| **Skills** | Construir o contexto | Capacidades empacotadas que o agente pode invocar |
| **Tool guidance** | Interface de ferramentas | Como as ferramentas são descritas ao modelo e como ele aprende a usá-las |
| **Scripts** | Interface de ferramentas | O código de apoio que prepara e processa o trabalho |
| **Subagents** | Delegação | Como o trabalho se divide entre agentes delegados |
| **Plugins** | Middleware | O comportamento transversal dentro do loop de ferramentas |

### O ciclo Propose–Probe–Compose

| Passo | Quem age | O que faz |
|---|---|---|
| **Propose** | O editor de harness | Propõe mudanças sobre os 8 componentes: de dois tipos, **invariantes** (valem para qualquer tarefa) e **de domínio** (específicas desta classe de tarefa) |
| **Probe** | Os executores candidatos | Cada proposta é executada sobre **a mesma tarefa**; não é preciso ter a resposta correta nem um verificador com a solução |
| **Compose** | O compositor | Compara as execuções entre si —o **contraste de comportamento**— e monta o harness refinado com o que demonstrou funcionar |

O orçamento de recursão é o `--k`: com `--k 1` há um único passo Propose–Probe–Compose. Há duas maneiras de escalar:

| Modo | Como escala | Numa frase |
|---|---|---|
| `--mode parallel` | Testa K candidatos gerais e K de domínio, e produz K composições a partir da evidência compartilhada | Explora mais largo |
| `--mode sequential` | Encadeia K passos Propose–Probe–Compose levando o harness refinado e a evidência acumulada | Refina mais fundo |

### Executores e benchmarks

O executor padrão é o **OpenCode**; com `--substrate cx` usa-se o **Codex**. Os resultados publicados correspondem a um único passo do Harness² sobre os harness-base (modelos GLM-5 e Gemini 3.5 Flash), segundo o paper e o README do projeto:

| Benchmark | O que mede | Instalação | Melhora publicada |
|---|---|---|---|
| **JobBench** | Trabalho profissional | `harness2-setup jb` | +4,5 pontos |
| **LAB** | Trabalho jurídico | `harness2-setup lab` | +11,8 pontos percentuais na taxa de aprovação parcial |
| **WorkBuddy-Bench** | Escritório, web e código | `harness2-setup wb` | Até +11,5 pontos |

### Comece pela demo local (sem gastar com API)

A demo roda os dois modos de recursão com editores determinísticos e execuções Python reais, **sem credenciais de modelo**:

```bash
# Linux · Python 3.12 · uv
uv venv --python 3.12 .venv
source .venv/bin/activate
uv pip install -e '.[render]'
harness2-demo --output runs/demo --k 2
```

Os benchmarks com modelo custam: pedem Node.js 20+, Google Cloud CLI e um projeto Vertex AI no `.env`; o LAB ainda precisa de Pandoc e Poppler, e o WorkBuddy-Bench, de Docker. Comece com `--max-tasks 1` e use `--dry-run` para ver quais tarefas seriam escolhidas sem chamar o modelo. Os resultados ficam em `runs/`, e repetir o mesmo comando retoma de onde parou.

### Harness² frente ao ECC, numa linha

O ECC é um pacote de otimizações de harness reunidas à mão por pessoas: instala-se como está e não muda entre tarefas (estático). O Harness² gera, testa e compõe as mudanças automaticamente em cada tarefa nova, e acumula o que aprendeu (dinâmico).

### ⚠️ Erros típicos

- **Julgar com outra configuração.** `harness2-judge` e `harness2-report` devem usar o mesmo benchmark, executor, subconjunto, `--k`, `--mode` e semente da rodada de refinamento; senão, os números não são comparáveis.
- **Avaliar com a rede aberta.** O executor tem acesso web sem restrições e o Codex roda com a sandbox desativada: as rubricas públicas dos benchmarks podem estar online. Configure as restrições de rede antes de avaliar, ou o agente pode «encontrar» as respostas em vez de resolver a tarefa.
- **Queimar orçamento sem ver a demo.** Sem ter visto o ciclo funcionar localmente com a demo gratuita, é fácil gastar chamadas de modelo numa seleção de tarefas errada.

---

## 🇬🇧 English summary

One-page cheat sheet (Spanish + Portuguese) for **Harness²** (google-research): recursive agent harnessing with model weights frozen. A harness is everything around a model — prompts, tools, memory, skills, orchestration — and Harness² improves it automatically on tasks the harness designer never saw. It exposes eight editable components in four runtime roles (system prompt, guardrails, memory, and skills construct context; tool guidance and scripts define the tool interface; subagents orchestrate delegation; plugins act as middleware). The task-level loop is Propose–Probe–Compose: an editor proposes invariant and domain edits, candidates are probed on the same task with no ground-truth feedback, and a composer builds the refined harness from behavioral contrasts between executions. Above it, a persistent improvement harness carries the adopted harness and evidence-supported lessons into the next task. Recursion scales in parallel (wider exploration) or sequential (deeper refinement) via `--k`; executors are OpenCode (default) and Codex. Published single-step gains over base harnesses (GLM-5, Gemini 3.5 Flash): up to +11.5 points on WorkBuddy-Bench, +4.5 on JobBench, +11.8 percentage points on LAB's partial pass rate. The sheet leads with the credential-free local demo (`harness2-demo`, Linux + Python 3.12 + uv) before any API spend, contrasts Harness² with the hand-assembled, static ECC optimization pack, and lists the typical mistakes: judging with a different configuration than the refinement run, and evaluating with unrestricted web access (rubrics may be online). Original synthesis by the author from the project's public documentation; not an official document and not a translation.

**Author:** Guangyi Zhao · October 2026 · MIT License
