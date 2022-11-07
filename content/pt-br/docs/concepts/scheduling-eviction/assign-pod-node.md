---
reviewers:
- davidopp
- kevin-wangzefeng
- bsalamat
title: Atribuindo Pods à Nodes
content_type: concept
weight: 20
---


<!-- overview -->

Você pode restringir um {{< glossary_tooltip text="Pod" term_id="pod" >}} e então ele estará _restrito_ a executar em um {{< glossary_tooltip text="node(s)" term_id="node" >}} específico, ou a executar _preferencialmente_ em nodes específicos. Exister algumas formas de fazer isso e todas abordagens recomendadas usam [label selectors](/docs/concepts/overview/working-with-objects/labels/) para facilitar a seleção. Geralmente, você não precisa configurar nenhuma restrição; o {{< glossary_tooltip text="scheduler" term_id="kube-scheduler" >}} irá automaticamente fazer uma alocação razoável (por exemplo, espalhar seus Pods através dos nodes para não colocar Pods em um node com recursos livres suficientes).
Entretanto, exister algumas circunstâncias onde você pode querer controlar em qual node o Pod será executado, por exemplo, para garantir que o Pod esteja em um node com SSD conectado, ou para alocar próximo Pods de dois serviços diferentes que se comunicam muito na mesma zona de disponibilidade.

<!-- body -->

Você pode usar qualquer um dos métodos a seguir para escolher onde o Kubernetes aloca Pods específicos:

  * [nodeSelector](#nodeselector) campo que coincide com [node labels](#built-in-node-labels)
  * [Affinity and anti-affinity](#affinity-and-anti-affinity)
  * [nodeName](#nodename) campo
  * [Pod topology spread constraints](#pod-topology-spread-constraints)

## Node labels {#built-in-node-labels}

Como muitos outros objetos Kubernetes, nodes possuem [labels](/docs/concepts/overview/working-with-objects/labels/). Você pode [anexar labels manualmente](/docs/tasks/configure-pod-container/assign-pods-nodes/#add-a-label-to-a-node).
Kubernetes também popula um grupo padrão de labels em todos os nodes de um cluster. Veja [Labels bem conhecidos, Annotations and Taints](/docs/reference/labels-annotations-taints/) para uma lista dos labels comuns de nodes.

{{<nota>}}
O conteúdo desses labels é específico  de cada provedor de nuvem e não há garantia de confiabilidade. 
Por exemplo, o conteúdo de `kubernetes.io/hostname` pode ser o mesmo que o nome do node em alguns ambientes e diferente em outros.
{{</nota>}}

### Isolamento/Restrição de Node

Adicionar labels à nodes te permite direcionar Pods para serem alocados em nodes ou grupos de nodes específicos. Você pode usar essa funcionalidade para garantir que Pods específicos sejam executados apenas em nodes com certo isolamento, segurança ou propriedades regulatórias.

Se você usa labels para isolamento de nodes, escolha chaves de labels que o {{<glossary_tooltip text="kubelet" term_id="kubelet">}} não consiga modificar. Isso previne um node comprimetido de configurar esses labels nele mesmo fazendo o schedule alocar cargas de trabalho num node comprometido.

O [`NodeRestriction` admission plugin](/docs/reference/access-authn-authz/admission-controllers/#noderestriction) previne o kubelet de configurar ou modificar labels com o prefixo `node-restriction.kubernetes.io/`.

Para usar esse prefixo de label para isomalento de node:

1. Garanta que está usando o [Node authorizer](/docs/reference/access-authn-authz/node/) e tem ativado o  admission plugin `NodeRestriction`.
2. Adicione labels com o prefixo `node-restriction.kubernetes.io/` aos seus nodes, e use esses labels em seu [node selectors](#nodeselector).
   Por exemplo, `example.com.node-restriction.kubernetes.io/fips=true` or `example.com.node-restriction.kubernetes.io/pci-dss=true`.