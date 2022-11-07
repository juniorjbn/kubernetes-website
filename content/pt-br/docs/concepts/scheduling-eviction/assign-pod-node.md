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