---
tags:
  - dashboard
  - emotional
---
## Numerical qualities tracker

This tracker shows the dynamics of the quality of the food i've been eating though out the day as well as the sleep quality

```tracker
searchType: frontmatter
searchTarget: sleep_quality, m_food_quality, d_food_quality, a_food_quality
datasetName: sleep quality, breakfast quality, lunch quality, supper quality
folder: "emotional-journal"
line:
	yMin: 0
    yMax: 2
    yAxisLocation: left
    yAxisLabel: from 0 to 2
    yAxisLabelColor: white, 
    lineColor: pink, orange, violet, green
    showLegend: true
    legendPosition: right
```

## Urges

This calendar shows the dynamics of how were my urges occur during the day. 

```dataview
TABLE
  choice(m_social_urge, "Y", " ") AS "M. Social",
  choice(d_social_urge, "Y", " ") AS "D. Social",
  choice(a_social_urge, "Y", " ") AS "A. Social",
  choice(m_work_urge, "Y", " ") AS "M. Work",
  choice(d_work_urge, "Y", " ") AS "D. Work",
  choice(a_work_urge, "Y", " ") AS "A. Work",
  choice(m_creative_urge, "Y", " ") AS "M. Creative",
  choice(d_creative_urge, "Y", " ") AS "D. Creative",
  choice(a_creative_urge, "Y", " ") AS "A. Creative",
  choice(lying, "Y", " ") AS "Was I lying"
FROM "emotional-journal"
WHERE contains(file.tags, "journal")
SORT file.name DESC
LIMIT 30
```



## D3 Global Graph Visualisation

```dataviewjs
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

try {
    const scriptPath = app.vault.adapter.getResourcePath("scripts/d3.v7.min.js");
    await loadScript(scriptPath);
    const allPages = dv.pages('"emotional-journal"');
    if (!allPages || allPages.length === 0) {
        dv.paragraph("'emotional-journal' folder is empty as fuck.");
        return;
    }

    const graphData = await parseNotesToGraph(allPages);
    if (!graphData.nodes || graphData.nodes.length === 0) {
        dv.paragraph("Can't find graph data.");
        return;
    }

    const nodeIds = new Set(graphData.nodes.map(n => n.id));
    const validEdges = graphData.edges.filter(edge => 
        nodeIds.has(edge.source) && nodeIds.has(edge.target)
    );

    const width = 1200;
    const height = 800;
    const svgContainer = dv.container.createEl('div', { style: `width: 100%; height: ${height}px; border: 1px solid grey; cursor: grab;` });
    const svg = d3.select(svgContainer).append("svg")
        .attr("width", "100%")
        .attr("height", height)
        .attr("viewBox", `0 0 ${width} ${height}`);

    const g = svg.append("g");

    const color = d3.scaleOrdinal(d3.schemeTableau10);

    const simulation = d3.forceSimulation(graphData.nodes)
        .force("link", d3.forceLink(validEdges).id(d => d.id).distance(150))
        .force("charge", d3.forceManyBody().strength(-400))
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
    
    node.filter(d => d.type !== 'time' && d.type !== 'event')
        .append("circle")
        .attr("r", 20)
        .attr("fill", d => color(d.type))
        .attr("stroke", "#fff")
        .attr("stroke-width", 1.5);

    node.filter(d => d.type === 'time' || d.type === 'event')
        .append("rect")
        .attr("width", 120).attr("height", 30)
        .attr("x", -60).attr("y", -15)
        .attr("rx", 5).attr("ry", 5)
        .attr("fill", "#555")
        .attr("stroke", "#fff")
        .attr("stroke-width", 1.5);
    
    node.append("text")
        .text(d => d.label)
        .attr("text-anchor", "middle")
        .attr("dy", ".35em")
        .style("fill", "white")
        .style("font-size", "10px")
        .style("text-shadow", "0 0 2px black, 0 0 2px black");

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

    const zoom = d3.zoom().on("zoom", (event) => {
        g.attr("transform", event.transform);
    });
    svg.call(zoom);

} catch (error) {
    dv.paragraph("ERROR: " + error.message);
    console.error(error);
}
```


