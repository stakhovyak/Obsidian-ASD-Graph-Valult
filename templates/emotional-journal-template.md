---
tags:
  - daily
  - emotional
  - journal
sleep_quality:
m_social_urge: false
m_work_urge: false
m_creative_urge: false
m_food_quality:
m_physical_activities:
d_social_urge: false
d_work_urge: false
d_creative_urge: false
d_food_quality:
d_physical_activities:
lying: false
symptoms:
medicines:
a_social_urge: false
a_work_urge: false
a_creative_urge: false
a_food_quality:
date:
---
## rules are
```
All the numeric properties could be:
- 0 - not at all / bad
- 1 - ok
- 2 - too much / awesome
Depending on the context!
```

use following categories:
- Время (Утро, День, Вечер, Ночь и тд)
- Состояние (Раздражение, Прокрастинация и тд)
- Погода (Облачно, Солнечно и тд)
- Вещество (Алкоголь, Никотин и тд)
- Ситуация (Лера злая, Мама психует, Потерял рюкзак и тд)
- Действие (Бегаю по улице под музыку, пою лежа на полу и тд)
- Чувство (Приятная неловкость, Грусть, Непонятное чувство закрытого окна и тд)
- Еда (Кофе, Чай, бургер в кфс)
- Вещь (сообщения в телеграм, изрисованная доска для маркеров и тд)
- Человек (Психотерапевт, Лера, Мама и тд)
---
# note

### *Morning (6:00 - 12:00)*

### *Day (12:00 - 18:00)*

### *Afternoon (18:00 - 23:00 or later)*


### local graph visualization


```dataviewjs
// Helplers
const loadScript = async (url) => {
    return new Promise((resolve, reject) => {
        if (document.querySelector(`script[src="${url}"]`)) { resolve(); return; }
        const script = document.createElement('script');
        script.src = url;
        script.onload = () => resolve();
        script.onerror = () => reject(new Error(`Script load error for ${url}`));
        document.head.appendChild(script);
    });
};

async function parseNotesToGraph(pages) {
    const nodes = new Map();
    const edges = [];

    function getCategoryType(label) {
        const parts = label.split(' - ');
        if (parts.length < 2) return 'category';
        const prefix = parts[0].toLowerCase().trim();
        switch (prefix) {
            case 'время': return 'time_cat';
            case 'состояние': return 'state';
            case 'погода': return 'weather';
            case 'вещество': return 'substance';
            case 'ситуация': return 'situation';
            case 'действие': return 'action';
            case 'чувство': return 'feeling';
            case 'еда': return 'food';
            case 'вещь': return 'thing';
            case 'человек': return 'person';
            case 'паттерн поведения': return 'pattern';
            default: return 'category';
        }
    }

    function addNode(id, label, type) {
        if (!id || !label) return null;
        const cleanedId = id.toLowerCase().trim()
            .replace(/\s+/g, '-')
            .replace(/[^a-zа-я0-9-]+/g, '');
        if (!cleanedId) return null;
        if (!nodes.has(cleanedId)) {
            nodes.set(cleanedId, { id: cleanedId, label: label, type: type });
        }
        return cleanedId;
    }

    for (const page of pages) {
        const content = await dv.io.load(page.file.path);
        if (!content) continue;
        const fileDate = page.file.name.match(/\d{4}-\d{2}-\d{2}/)?.[0] || page.file.name;
        let currentH3Id = null;
        let currentH4Id = null;
        let lastSourceNodeId = null;
        const lines = content.split('\n');

        for (const line of lines) {
            const h3Match = line.match(/^###\s+\*(.+?)\*/);
            const h4Match = line.match(/^####\s+(.+)/);

            if (h3Match && h3Match[1].trim()) {
                const h3Label = h3Match[1].trim();
                currentH3Id = addNode(`${fileDate}-${h3Label}`, h3Label, 'time');
                lastSourceNodeId = currentH3Id;
                currentH4Id = null;
            } else if (h4Match && h4Match[1].trim()) {
                let h4Text = h4Match[1].trim();
                const cleanH4Label = h4Text.replace(/\[\[.*?\]\]/g, '').replace(/\s{2,}/g, ' ').trim();
                if (cleanH4Label) {
                    currentH4Id = addNode(`${fileDate}-${cleanH4Label}`, cleanH4Label, 'event');
                    if (currentH3Id && currentH4Id) {
                        edges.push({ source: currentH3Id, target: currentH4Id });
                    }
                    lastSourceNodeId = currentH4Id;
                } else {
                    lastSourceNodeId = currentH3Id;
                }
            }

            const linkMatches = [...line.matchAll(/\[\[([^|\]]+)\]\]/g)];
            if (linkMatches.length > 0 && lastSourceNodeId) {
                for (const match of linkMatches) {
                    const linkLabel = match[1].trim();
                    if (!linkLabel) continue;
                    const linkType = getCategoryType(linkLabel);
                    const cleanLabel = linkLabel.includes(' - ') ? linkLabel.split(' - ').pop().trim() : linkLabel;
                    const linkId = addNode(linkLabel, cleanLabel, linkType);
                    if (linkId) {
                        edges.push({ source: lastSourceNodeId, target: linkId });
                    }
                }
            }
        }
    }
    return { nodes: Array.from(nodes.values()), edges: edges };
}

// Interactive graph main
try {
    const scriptPath = app.vault.adapter.getResourcePath("scripts/d3.v7.min.js");
    await loadScript(scriptPath);
    
    const graphData = await parseNotesToGraph([dv.current()]);
    if (!graphData.nodes || graphData.nodes.length === 0) {
        dv.paragraph("Данные для графа не найдены.");
        return;
    }

    const nodeIds = new Set(graphData.nodes.map(n => n.id));
    const validEdges = graphData.edges.filter(edge => 
        nodeIds.has(edge.source) && nodeIds.has(edge.target)
    );

    const width = 800;
    const height = 600;
    const svgContainer = dv.container.createEl('div', { style: `width: 100%; height: ${height}px; border: 1px solid grey; cursor: grab;` });
    const svg = d3.select(svgContainer).append("svg")
        .attr("width", "100%")
        .attr("height", height)
        .attr("viewBox", `0 0 ${width} ${height}`);

    const g = svg.append("g");

    // color palette
    const color = d3.scaleOrdinal(d3.schemeTableau10);

    const simulation = d3.forceSimulation(graphData.nodes)
        .force("link", d3.forceLink(validEdges).id(d => d.id).distance(120))
        .force("charge", d3.forceManyBody().strength(-300))
        .force("center", d3.forceCenter(width / 2, height / 2));

    const link = g.append("g")
        .attr("stroke", "#999")
        .attr("stroke-opacity", 0.6)
        .selectAll("line")
        .data(validEdges)
        .join("line")
        .attr("stroke-width", 1.5);

    const node = g.append("g")
        .selectAll("g")
        .data(graphData.nodes)
        .join("g");
    
    // categories design
    node.filter(d => d.type !== 'time' && d.type !== 'event')
        .append("circle")
        .attr("r", 20)
        .attr("fill", d => color(d.type)) 
        .attr("stroke", "#fff")
        .attr("stroke-width", 1.5);

    // rectangulars for headers
    node.filter(d => d.type === 'time' || d.type === 'event')
        .append("rect")
        .attr("width", 120).attr("height", 30)
        .attr("x", -60).attr("y", -15) 
        .attr("rx", 5).attr("ry", 5) 
        .attr("fill", "#555") 
        .attr("stroke", "#fff")
        .attr("stroke-width", 1.5);
    
    // labels
    node.append("text")
        .text(d => d.label)
        .attr("text-anchor", "middle")
        .attr("dy", ".35em")
        .style("fill", "white")
        .style("font-size", "14px")
        .style("text-shadow", "0 0 2px black, 0 0 2px black");

    // Moving nodes capability
    function drag(simulation) {
      function dragstarted(event) {
        if (!event.active) simulation.alphaTarget(0.3).restart();
        event.subject.fx = event.subject.x;
        event.subject.fy = event.subject.y;
      }
      function dragged(event) {
        event.subject.fx = event.x;
        event.subject.fy = event.y;
      }
      function dragended(event) {
        if (!event.active) simulation.alphaTarget(0);
        event.subject.fx = null;
        event.subject.fy = null;
      }
      return d3.drag().on("start", dragstarted).on("drag", dragged).on("end", dragended);
    }
    node.call(drag(simulation));

    simulation.on("tick", () => {
        link
            .attr("x1", d => d.source.x)
            .attr("y1", d => d.source.y)
            .attr("x2", d => d.target.x)
            .attr("y2", d => d.target.y);
        node
            .attr("transform", d => `translate(${d.x},${d.y})`);
    });

    // interactivity
    const zoom = d3.zoom().on("zoom", (event) => {
        g.attr("transform", event.transform);
    });
    svg.call(zoom);

} catch (error) {
    dv.paragraph("ERROR: " + error.message);
    console.error(error);
}
```

