<!-- overview -->

Você pode restringir um {{< glossary_tooltip text="Pod" term_id="pod" >}} e então ele estará _restrito_ a executar em um {{< glossary_tooltip text="node(s)" term_id="node" >}} específico, ou a executar _preferencialmente_ em nodes específicos. Existem algumas formas de fazer isso e todas as abordagens recomendadas usam [label selectors](/docs/concepts/overview/working-with-objects/labels/) para facilitar a seleção. Geralmente, você não precisa configurar nenhuma restrição; o {{< glossary_tooltip text="scheduler" term_id="kube-scheduler" >}} irá automaticamente fazer uma alocação razoável (por exemplo, espalhar seus Pods através dos nodes para não colocar Pods em um node com recursos livres suficientes).
Entretanto, existem algumas circunstâncias onde você pode querer controlar em qual node o Pod será executado, por exemplo, para garantir que o Pod esteja em um node com SSD conectado, ou para alocar próximo Pods de dois serviços diferentes que se comunicam muito na mesma zona de disponibilidade.

<!-- body -->

Você pode usar qualquer um dos métodos a seguir para escolher onde o Kubernetes aloca Pods específicos:

  * [nodeSelector](#nodeselector) campo que coincide com [node labels](#built-in-node-labels)
  * [Affinity and anti-affinity](#affinity-and-anti-affinity)
  * [nodeName](#nodename) campo
  * [Pod topology spread constraints](#pod-topology-spread-constraints)

## Node labels nativos {#built-in-node-labels}

Como muitos outros objetos Kubernetes, nodes possuem [labels](/docs/concepts/overview/working-with-objects/labels/). Você pode [anexar labels manualmente](/docs/tasks/configure-pod-container/assign-pods-nodes/#add-a-label-to-a-node).
Kubernetes também popula um grupo padrão de labels em todos os nodes de um cluster. Veja [Labels bem conhecidos, Annotations and Taints](/docs/reference/labels-annotations-taints/) para uma lista dos labels comuns de nodes.

{{<nota>}}
O conteúdo desses labels é específico  de cada provedor de nuvem e não há garantia de confiabilidade. 
Por exemplo, o conteúdo de `kubernetes.io/hostname` pode ser o mesmo que o nome do node em alguns ambientes e diferente em outros.
{{</nota>}}

### Isolamento/Restrição de Node

Adicionar labels a nodes te permite direcionar Pods para serem alocados em nodes ou grupos de nodes específicos. Você pode usar essa funcionalidade para garantir que Pods específicos sejam executados apenas em nodes com certo isolamento, segurança ou propriedades regulatórias.

Se você usa labels para isolamento de nodes, escolha chaves de labels que o {{<glossary_tooltip text="kubelet" term_id="kubelet">}} não consiga modificar. Isso previne um node comprometido de configurar esses labels nele mesmo fazendo o schedule alocar cargas de trabalho num node comprometido.

O [`NodeRestriction` admission plugin](/docs/reference/access-authn-authz/admission-controllers/#noderestriction) previne o kubelet de configurar ou modificar labels com o prefixo `node-restriction.kubernetes.io/`.

Para usar esse prefixo de label para isolamento de node:

1. Garanta que está usando o [Node authorizer](/docs/reference/access-authn-authz/node/) e tem ativado o  admission plugin `NodeRestriction`.
2. Adicione labels com o prefixo `node-restriction.kubernetes.io/` aos seus nodes, e use esses labels em seu [node selectors](#nodeselector).
   Por exemplo, `exemplo.com.node-restrito.kubernetes.io/pci-dss=true` ou `example.com.node-restriction.kubernetes.io/fips=true`.

## nodeSelector

`nodeSelector`é a forma mais simples recomendada para seleção de restrição em nodes.
Você pode adicionar o campo `nodeSelector` à especificação de seu Pod e definir o [label do node](#built-in-node-labels) de destino que você deseja tenha.
O Kubernetes apenas aloca o Pod em nodes que tenham os labels que você especificou.

Veja [Assign Pods to Nodes](/docs/tasks/configure-pod-container/assign-pods-nodes) para mais informações.

## Affinity e anti-affinity

``nodeSeletor`é a forma mais simples de restringir Pods à nodes com labels específicos, Affinity e anti-affinity expandem os tipos de restrições que voê pode definir. Alguns dos benefícios que affinity e anti-affinity incluem:

* A linguagem de affinity/anti-affinity é mais expressiva. `nodeSelector`seleciona apenas nodes com todos os labels especificados. Affinity/anti-affinity te dá maior controle em cima da lógica de seleção.
* Você pode indicar uma regra como *soft* ou *preferred*, então o scheduler ainda alocará um Pod, mesmo que não encontre um node adequado.
* Você pode restringir um Pod usando labels em outros Pods rodando em um node (ou outros domínios de topologia), em vez de usar apenas labels em nodes, o que te permite definir regras para quais Pods podem compartilhar o mesmo node.

Affinity consiste em dois tipos:

* Funções de *Node affinity* como o campo `nodeSelector, mas isso é mais expressivo e te permite especificar regras flexíveis.
* *Inter-pod affinity/anti-affinity* te permite restringir Pods com labels em outros Pods.

### Node Affinity

Node Affinity é conceitualmente similar a `nodeSelector`, permitindo que você restrinja em quais nodes seu Pod pode ser alocado com base nos labels do node. Existem dois tipos de node affinity:

  * `requiredDuringSchedulingIgnoredDuringExecution`: O scheduler não poderá alocar o Pod a menos que a regra seja satisfeita. Isso funciona como `nodeSelector`, mas com uma sintaxe mais expressiva.
  * `preferredDuringSchedulingIgnoredDuringExecution`: O scheduler tenta encontrar um node que satisfaça a regra. Se o node escolhido não estiver disponível, o scheduler ainda tentará alocar o Pod.

{{<note>}}
Nos tipos anteriores, `IgnoredDuringExecution` significa que se o label do node mudar depois que o Kubernetes alocar o Pod, o Pod continua a ser executado.
{{</note>}}

Você pode especificar node affinity usando o campo `.spec.affinity.nodeAffinity` na configuração do Pod.

Por exemplo, considere a configuração de Pod a seguir:

{{< codenew file="pods/pod-with-node-affinity.yaml" >}}

Nesse exemplo, as seguintes regras se aplicam:

  * O node *precisa* ter o label com a chave `topology.kubernetes.io/zone` e o valor desse label *precisa* ser `antarctica-east1` ou `antarctica-west1`.
  * O node *preferencialmente* possui a label com a chave `another-node-label-key` e o valor `another-node-label-key`.

Você pode usar o campo `operator`para especificar um operador lógico para o Kubernetes utilizar quando for interpretar as regras. Você pode usar `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`e `Lt`. (Em português Contém, NãoContém, Existe, NãoExiste, MaiorQue e MenorQue)

`NotIn` e `DoesNotExist`te permite definir um comportamento de anti-affinity do node.
De forma alternativa, você pode usar [node taints](/docs/concepts/scheduling-eviction/taint-and-toleration/) para repelir Pods de nodes específicos.

{{<node>}}
Se você especificar ambos `nodeSelector` e `nodeAffinity`, *ambos* precisam ser satisfeitos para que o Pod seja alocado no node.

Se você especificar multiplos `nodeSelectorTerms` associados com tipos de `nodeAffinity`, então o Pod pode ser alocado no node apenas se todas as `matchExpressions` forem satisfeitas.
{{</node>}}

Veja [Assign Pods to Nodes using Node Affinity](/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/) para mais informações.

#### Peso em node affinity

