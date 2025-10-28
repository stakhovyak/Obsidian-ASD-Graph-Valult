# My Psychoanalytic Journal (Obsidian with D3 visualizations)

This vault is adapted for journaling in **Russian**. While this README is in English, the categorical tagging system is built around Russian prefixes (e.g., `[[Состояние - Тревога]]`).

-----

## To do

- [ ] add search by word for category nodes in graph
- [ ] make clicking on the node with category open up it's note to see description and backlinks
- [ ] make clicking on certain context (gray rectangular) open up the daily note it's being mentioned in

## Concept

this system is about logging metrics in a free-form, narrative way. It's built on the understanding that the most honest insights come from unfiltered thoughts. This format allows you to write down absolutely anything that comes to mind—chaotic thoughts, nonsensical ideas, profanity, or deeply personal events.

The true value lies in the tagged metadata, not just the prose.

The generated graph represents the *metrics* of your mental state—the connections and frequencies of moods, situations, and behaviors. This high-level view can be incredibly useful. For instance, you could show the visual graph to a therapist or specialist. They could identify valuable patterns and gain insights for their work without needing to read the raw, intimate, and sometimes chaotic details of the notes themselves.

-----

## How It Works: The Structure

The vault relies on a structure within your daily notes. Adhering to this format is crucial for the automated scripts to build your graph correctly. All daily notes should be placed in the `emotional-journal` folder. Use whether feelings, factors as your links, but it's very important to follow the established prefixes and use the correct levels of the headers!

### **1. Time Blocks (H3 Headers)**

Divide your day into logical periods using Level 3 Headers with asterisks.
`### *Morning (6:00 - 12:00)*`
`### *Day (12:00 - 18:00)*`

### **2. Events & Situations (H4 Headers)**

Under each time block, describe specific events, actions, or significant thoughts using Level 4 Headers.
`#### Woke up feeling anxious`
`#### Thought about the conversation with Lena`

### **3. Categorical Links (The Core)**

This is the most important part. Within the text under any header, link to specific categories using wikilinks `[[...]]`. **These must be in Russian.** The script uses the prefix (e.g., "Состояние - ") to automatically color and shape the nodes on the graph.

**Use the following categories:**

  * **Время**: `[[Время - Утро]]`
  * **Состояние**: `[[Состояние - Раздражение]]`, `[[Состояние - Прокрастинация]]`
  * **Погода**: `[[Погода - Облачно]]`
  * **Вещество**: `[[Вещество - Никотин]]`, `[[Вещество - Кофеин]]`
  * **Ситуация**: `[[Ситуация - Лена злая]]`, `[[Ситуация - Потерял рюкзак]]`
  * **Действие**: `[[Действие - Слушаю музыку]]`, `[[Действие - Мастурбация]]`
  * **Чувство**: `[[Чувство - Грусть]]`, `[[Чувство - Приятная неловкость]]`
  * **Еда**: `[[Еда - Кофе]]`, `[[Еда - Бургер]]`
  * **Вещь**: `[[Вещь - Сообщения в телеграм]]`
  * **Человек**: `[[Человек - Психотерапевт]]`, `[[Человек - Лена]]`
  * **Паттерн Поведения**: `[[Паттерн Поведения - Пытаюсь нормально себя вести]]`
  
The letters' register doesn't matter though.

#### **Example Entry:**

```markdown
### *Morning (6:00 - 12:00)*
Felt a familiar sense of [[Состояние - Раздражение]] today. Probably connected to [[Состояние - Упадок сил]]. The weather is [[Погода - Облачно]], which I like. Used some [[Вещество - Никотин]].

#### When Lena left for work
[[Ситуация - Лены нет дома]]. There was a strange feeling of [[Чувство - Облегчение]] mixed with [[Чувство - Грусть]].
```

### Make sure you have Tracker and Dataview community plugins installed.