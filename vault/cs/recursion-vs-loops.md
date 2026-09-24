---
title: Recursion vs Loops
tags: [concept, pattern, tradeoff]
date: 2026-04-12
---

# Рекурсия vs циклы

moc: [[fp-patterns-moc]]
next: [[pure-functions]]

---

```
findAll(root)
├── root matched? → [root]
├── findAll(child1)
│   ├── findAll(child1.1) → []
│   └── findAll(child1.2) → [child1.2]
└── findAll(child2)
    └── findAll(child2.1) → [child2.1]
результат: [child1.2, child2.1]
```

В чистом ФП нет `for`/`while` — только рекурсия. В мейнстриме полезна для деревьев и вложенных структур.

```ts
// обход дерева — рекурсия естественнее цикла
type TreeNode = { value: string; children: TreeNode[] };

function findAll(node: TreeNode, predicate: (n: TreeNode) => boolean): TreeNode[] {
    const result = predicate(node) ? [node] : [];
    return [...result, ...node.children.flatMap(c => findAll(c, predicate))];
}
```

## Когда что

- **Деревья** (DOM, AST, файловая система) — рекурсия единственный читаемый способ
- **Плоские данные** — обычные циклы проще и эффективнее
