---
title: SVM (Support Vector Machine)
tags: [concept, ml, classification]
date: 2026-04-10
---

# SVM (Support Vector Machine)

moc: [[ml-moc]]
next: [[logistic-regression]] [[random-forest]]

---

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 520 360" font-family="system-ui, -apple-system, sans-serif">
  <defs>
    <marker id="arrow" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6" fill="#888"/>
    </marker>
  </defs>

  <!-- Background -->
  <rect width="520" height="360" rx="12" fill="#1e1e2e"/>

  <!-- Class A region (left) -->
  <rect x="30" y="30" width="190" height="300" rx="8" fill="#2a1a3a" opacity="0.5"/>
  <!-- Class B region (right) -->
  <rect x="300" y="30" width="190" height="300" rx="8" fill="#1a2a3a" opacity="0.5"/>

  <!-- Margin zone -->
  <rect x="220" y="30" width="80" height="300" rx="0" fill="#333355" opacity="0.35"/>

  <!-- Margin boundary lines (dashed) -->
  <line x1="220" y1="30" x2="220" y2="330" stroke="#7c7caa" stroke-width="1.5" stroke-dasharray="6,4"/>
  <line x1="300" y1="30" x2="300" y2="330" stroke="#7c7caa" stroke-width="1.5" stroke-dasharray="6,4"/>

  <!-- Decision boundary (center, solid) -->
  <line x1="260" y1="25" x2="260" y2="335" stroke="#e0e040" stroke-width="2.5"/>

  <!-- Margin arrows -->
  <line x1="222" y1="345" x2="258" y2="345" stroke="#888" stroke-width="1.2" marker-end="url(#arrow)"/>
  <line x1="298" y1="345" x2="262" y2="345" stroke="#888" stroke-width="1.2" marker-end="url(#arrow)"/>
  <text x="260" y="355" text-anchor="middle" fill="#aaa" font-size="10">margin</text>

  <!-- Class A points -->
  <circle cx="70"  cy="80"  r="8" fill="#c77dff" opacity="0.7"/>
  <circle cx="120" cy="110" r="8" fill="#c77dff" opacity="0.7"/>
  <circle cx="55"  cy="160" r="8" fill="#c77dff" opacity="0.7"/>
  <circle cx="100" cy="200" r="8" fill="#c77dff" opacity="0.7"/>
  <circle cx="140" cy="150" r="8" fill="#c77dff" opacity="0.7"/>
  <circle cx="80"  cy="260" r="8" fill="#c77dff" opacity="0.7"/>
  <circle cx="130" cy="280" r="8" fill="#c77dff" opacity="0.7"/>
  <circle cx="160" cy="230" r="8" fill="#c77dff" opacity="0.7"/>
  <circle cx="60"  cy="310" r="8" fill="#c77dff" opacity="0.7"/>
  <circle cx="110" cy="60"  r="8" fill="#c77dff" opacity="0.7"/>

  <!-- Support vectors class A (highlighted) -->
  <circle cx="195" cy="120" r="9" fill="none" stroke="#c77dff" stroke-width="2.5"/>
  <circle cx="195" cy="120" r="5" fill="#c77dff"/>
  <circle cx="185" cy="240" r="9" fill="none" stroke="#c77dff" stroke-width="2.5"/>
  <circle cx="185" cy="240" r="5" fill="#c77dff"/>
  <circle cx="200" cy="180" r="9" fill="none" stroke="#c77dff" stroke-width="2.5"/>
  <circle cx="200" cy="180" r="5" fill="#c77dff"/>

  <!-- Class B points -->
  <circle cx="430" cy="70"  r="8" fill="#56c8f5" opacity="0.7"/>
  <circle cx="380" cy="120" r="8" fill="#56c8f5" opacity="0.7"/>
  <circle cx="450" cy="160" r="8" fill="#56c8f5" opacity="0.7"/>
  <circle cx="400" cy="210" r="8" fill="#56c8f5" opacity="0.7"/>
  <circle cx="360" cy="260" r="8" fill="#56c8f5" opacity="0.7"/>
  <circle cx="420" cy="290" r="8" fill="#56c8f5" opacity="0.7"/>
  <circle cx="460" cy="240" r="8" fill="#56c8f5" opacity="0.7"/>
  <circle cx="380" cy="310" r="8" fill="#56c8f5" opacity="0.7"/>
  <circle cx="440" cy="100" r="8" fill="#56c8f5" opacity="0.7"/>
  <circle cx="410" cy="50"  r="8" fill="#56c8f5" opacity="0.7"/>

  <!-- Support vectors class B (highlighted) -->
  <circle cx="325" cy="100" r="9" fill="none" stroke="#56c8f5" stroke-width="2.5"/>
  <circle cx="325" cy="100" r="5" fill="#56c8f5"/>
  <circle cx="320" cy="200" r="9" fill="none" stroke="#56c8f5" stroke-width="2.5"/>
  <circle cx="320" cy="200" r="5" fill="#56c8f5"/>
  <circle cx="330" cy="280" r="9" fill="none" stroke="#56c8f5" stroke-width="2.5"/>
  <circle cx="330" cy="280" r="5" fill="#56c8f5"/>

  <!-- Labels -->
  <text x="90"  y="48" text-anchor="middle" fill="#c77dff" font-size="13" font-weight="600">Класс A</text>
  <text x="410" y="48" text-anchor="middle" fill="#56c8f5" font-size="13" font-weight="600">Класс B</text>

  <!-- Decision boundary label -->
  <text x="260" y="18" text-anchor="middle" fill="#e0e040" font-size="11" font-weight="500">decision boundary</text>

  <!-- Support vector annotation -->
  <line x1="195" y1="120" x2="195" y2="75" stroke="#aaa" stroke-width="0.8" stroke-dasharray="3,2"/>
  <text x="195" y="68" text-anchor="middle" fill="#ddd" font-size="9">support vector</text>

  <line x1="325" y1="100" x2="325" y2="60" stroke="#aaa" stroke-width="0.8" stroke-dasharray="3,2"/>
  <text x="325" y="53" text-anchor="middle" fill="#ddd" font-size="9">support vector</text>
</svg>

Как логрег, но ищет плоскость с **максимальным зазором** (margin) от ближайших точек обоих классов. Ближайшие точки к границе — support vectors (опорные векторы).

Чем шире зазор, тем увереннее модель на новых данных.

**Kernel trick** — бонус SVM. Если данные не разделяются линейно, SVM "поднимает" их в более высокое измерение, где разделение возможно.
