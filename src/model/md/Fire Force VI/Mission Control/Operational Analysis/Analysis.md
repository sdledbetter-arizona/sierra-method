---
ontology: https://fireforce6.github.io/mission-control/bundle
---

# Assignment 5 Analysis

These checks assess how operational entities are assigned capabilities. An empty conformance or orphan result is meaningful: it means the tested missing relationship was not found.

## Conformance

**Rule:** Every operational entity is allocated at least one capability.

```table
PREFIX entity: <https://www.modelware.io/sierra/entity#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>
PREFIX mission: <https://www.modelware.io/sierra/mission#>

SELECT ?entity WHERE {
  ?entity a entity:Entity .
  FILTER NOT EXISTS { ?entity a stakeholder:Stakeholder }
  FILTER NOT EXISTS {
    ?entity entity:hasCapability ?capability .
    ?capability a mission:Capability
  }
}
```

No rows means every operational entity has at least one capability allocation.

## Near Miss

Which capabilities are represented but assigned to only one operational entity? These are concentration candidates, not automatic defects.

```table
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>
PREFIX mission: <https://www.modelware.io/sierra/mission#>

SELECT ?capability ?label (COUNT(DISTINCT ?entity) AS ?ownerCount) WHERE {
  ?entity a entity:Entity ; entity:hasCapability ?capability .
  FILTER NOT EXISTS { ?entity a stakeholder:Stakeholder }
  ?capability a mission:Capability .
  OPTIONAL { ?capability rdfs:label ?label }
}
GROUP BY ?capability ?label
HAVING(COUNT(DISTINCT ?entity) = 1)
ORDER BY ?capability
```

## Orphans

Are any capabilities missing an allocation to every operational entity? This check uses `FILTER NOT EXISTS`; no rows is a clean result, not missing evidence.

```table
PREFIX entity: <https://www.modelware.io/sierra/entity#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>
PREFIX mission: <https://www.modelware.io/sierra/mission#>

SELECT ?capability WHERE {
  ?capability a mission:Capability .
  FILTER NOT EXISTS {
    ?owner a entity:Entity ; entity:hasCapability ?capability .
    FILTER NOT EXISTS { ?owner a stakeholder:Stakeholder }
  }
}
```

## View Graph

The `CONSTRUCT` reshapes allocations as diagram nodes and edges for visual inspection.

```diagram
---
stylesheet:
  - selector: node.entity
    style:
      fill: lightblue
  - selector: node.capability
    style:
      fill: lightgreen
  - selector: diagram
    style:
      layout:
        type: dagre
        rankdir: LR
---
PREFIX diagram: <http://opencaesar.io/diagram#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>
PREFIX mission: <https://www.modelware.io/sierra/mission#>

CONSTRUCT {
  ?entity a diagram:Node ; diagram:text ?entityLabel ; diagram:class "entity" .
  ?capability a diagram:Node ; diagram:text ?capabilityLabel ; diagram:class "capability" .
  ?edge a diagram:Edge ; diagram:source ?entity ; diagram:target ?capability .
}
WHERE {
  ?entity a entity:Entity ; entity:hasCapability ?capability .
  FILTER NOT EXISTS { ?entity a stakeholder:Stakeholder }
  ?capability a mission:Capability .
  BIND(REPLACE(STR(?entity), "^.*[#/]", "") AS ?entityLabel)
  BIND(REPLACE(STR(?capability), "^.*[#/]", "") AS ?capabilityLabel)
  BIND(IRI(CONCAT("https://fireforce6.github.io/mission-control/analysis/edge/", MD5(CONCAT(STR(?entity), STR(?capability))))) AS ?edge)
}
```

## Question, Evidence, Interpretation

**Question:** Which capabilities depend on a single operational entity?

**Evidence:** The near-miss result identifies `C1`, `C4`, and `C7`; the scripted view below computes their share of all allocated capabilities.

**Interpretation:** These capabilities have a single modeled owner. That is a concentration signal to review against the intended architecture, not proof that the allocation is wrong.

```python
include('src/method/py/utils.py')
import micropip
await micropip.install(['matplotlib'])
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

result = await query("""
  PREFIX entity: <https://www.modelware.io/sierra/entity#>
  PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>
  PREFIX mission: <https://www.modelware.io/sierra/mission#>
  SELECT ?capability (COUNT(DISTINCT ?entity) AS ?ownerCount) WHERE {
    ?entity a entity:Entity ; entity:hasCapability ?capability .
    FILTER NOT EXISTS { ?entity a stakeholder:Stakeholder }
    ?capability a mission:Capability .
  }
  GROUP BY ?capability
  HAVING(COUNT(DISTINCT ?entity) = 1)
  ORDER BY ?capability
""")

rows = result['rows']
labels = [row['capability'].rsplit('#', 1)[-1] for row in rows]
owner_counts = [int(row['ownerCount']) for row in rows]

fig, ax = plt.subplots(figsize=(6, max(2.2, len(labels) * 0.5)))
ax.barh(labels, owner_counts, color='#d97742')
ax.set_xlim(0, 2)
ax.set_xticks([0, 1, 2])
ax.set_xlabel('Operational entity owners')
ax.set_title(f'Near-miss capabilities: {len(labels)} single-owner allocations')
ax.spines[['top', 'right', 'left']].set_visible(False)
ax.tick_params(axis='y', length=0)
plt.tight_layout()
display(image_html(fig))
```

## Reusable Coverage Matrix

The matrix template uses `COALESCE` to retain explicit zeros for unassigned entity/capability pairs.

```compose
template: https://www.modelware.io/sierra/operational-analysis/capability-coverage
```