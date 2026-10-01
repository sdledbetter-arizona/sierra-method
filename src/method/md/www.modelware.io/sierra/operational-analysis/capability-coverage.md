---
template:
  id: https://www.modelware.io/sierra/operational-analysis/capability-coverage
  name: "Capability Coverage Matrix"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Entity / Capability Coverage

Each cell reports whether an operational entity is assigned a capability: `1` means an explicit allocation and `0` means none is modeled. A zero is a gap to assess, not an assertion that the entity should have that capability.

```matrix
---
rowColumnLabel: Entity / Capability
stylesheet:
  - selector: cell [Number(value) === 0]
    style:
      background-color: mistyrose
  - selector: cell [Number(value) === 1]
    style:
      background-color: lightgreen
---
PREFIX entity: <https://www.modelware.io/sierra/entity#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>
PREFIX mission: <https://www.modelware.io/sierra/mission#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  ?row a entity:Entity .
  FILTER NOT EXISTS { ?row a stakeholder:Stakeholder }
  ?column a mission:Capability .

  OPTIONAL {
    SELECT ?row ?column (COUNT(DISTINCT ?row) AS ?n)
    WHERE {
      ?row a entity:Entity ; entity:hasCapability ?column .
      FILTER NOT EXISTS { ?row a stakeholder:Stakeholder }
      ?column a mission:Capability .
    }
    GROUP BY ?row ?column
  }
}
ORDER BY ?row ?column
```