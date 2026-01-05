# FORMULACIÓN MATEMÁTICA Y PSEUDOCÓDIGO (ANEXO TÉCNICO)

> Nota de interoperabilidad: en Markdown estándar, las fórmulas pueden renderizarse con MathJax/KaTeX (GitHub no siempre renderiza LaTeX inline/bloque en README sin extensiones).  
> Aun así, se dejan en notación LaTeX por rigor y trazabilidad con el paper/notebook.

---

## 0) Datos y normalización

### Dataset

\[
X=\{x_i\}_{i=1}^N,\quad x_i\in\mathbb{R}^d
\]

### Z-score (por feature)

Sea \(\mu_f\) y \(\sigma_f\) la media y desviación típica de la feature \(f\) en todo el dataset:

\[
\tilde{x}_{i,f}=\frac{x_{i,f}-\mu_f}{\sigma_f}
\]

Denote \(\tilde{x}_i\in\mathbb{R}^d\) al vector normalizado.

---

## 1) Distancia semántica y umbral \(\tau\)

### Distancia (baseline, comparable a k-means)

\[
D(\tilde{x}_i,\tilde{x}_j)=\|\tilde{x}_i-\tilde{x}_j\|_2
\]

### Cálculo exacto de \(\tau\) (percentil 15%)

Construya el multiconjunto de distancias entre todos los pares:

\[
\mathcal{D}=\left\{D(\tilde{x}_i,\tilde{x}_j)\;|\;1\le i<j\le N\right\}
\]
con tamaño \(|\mathcal{D}|=\frac{N(N-1)}{2}\).

Entonces:

\[
\tau = \text{percentil}_{0.15}(\mathcal{D})
\]

**Implementación (opcional):** se permite estimar \(\tau\) mediante muestreo uniforme de pares con ratio \(\rho\in(0,1]\), manteniendo el caso exacto cuando \(\rho=1\).

---

## 2) Grid 2D como espacio operativo

### Tamaño del grid

\[
|G|=\lambda N,\quad \lambda=4
\]

Defina dimensiones discretas:

\[
W=\left\lceil \sqrt{|G|}\right\rceil,\quad
H=\left\lceil \frac{|G|}{W}\right\rceil
\]

y el grid:

\[
G=\{0,\dots,W-1\}\times\{0,\dots,H-1\}
\]

### Asignación inicial de objetos (placement)

Se asigna cada objeto a una celda libre, aleatoriamente:

\[
pos(i)\in G
\]

con la restricción: **1 objeto por celda**.

---

## 3) Balizas (beacons) y prototipos online

Sea el conjunto de balizas:

\[
B=\{b_k\}_{k=1}^{K}
\]

Cada baliza \(b_k\) tiene:

- posición en el grid: \(c_k\in G\)
- prototipo: \(\mu_k\in\mathbb{R}^d\)
- contador: \(n_k\in\mathbb{N}\)

### Inicialización de una baliza nueva (clúster nuevo)

Si se crea \(b_{K+1}\) a partir de un objeto \(i\):

\[
c_{K+1}\leftarrow pos(i),\quad
\mu_{K+1}\leftarrow \tilde{x}_i,\quad
n_{K+1}\leftarrow 1
\]

### Actualización online del prototipo (cuando se marca y asigna)

Si el objeto \(i\) se asigna al clúster \(k\):

\[
n_k \leftarrow n_k + 1
\]

\[
\mu_k \leftarrow \mu_k + \frac{1}{n_k}\left(\tilde{x}_i-\mu_k\right)
\]

---

## 4) Etiquetas (opsonización) y estado del objeto

Cada objeto \(i\) tiene una etiqueta:

\[
\ell(i)\in \{0,1,\dots,K\}
\]

donde \(\ell(i)=0\) significa “no etiquetado”.

Al asignar \(i\) al clúster \(k\):

\[
\ell(i)\leftarrow k
\]

Cada objeto \(i\) tiene un flag:
- `discarded(i) ∈ {False, True}` inicializado a `False`.

Cuando un Transportador deposita \(i\), se fija `discarded(i) ← True` y el objeto queda fuera de la dinámica.

Esta es la “marca estigmérgica” que coordina el sistema.

---

## 5) Vecindarios (objetos y balizas)

### Vecindario en grid (Chebyshev)

Defina la distancia de Chebyshev:

\[
d_\infty((x,y),(u,v))=\max(|x-u|,|y-v|)
\]

### Vecindario de radio \(r\) alrededor de una celda \(p\in G\)

\[
N_r(p)=\{q\in G\;|\; d_\infty(q,p)\le r\}
\]

- **Percepción de agentes (objetos):** \(r=1\) (Moore 3×3).
- **Escucha de balizas:** \(r=r_b\) (hiperparámetro experimental).

---

## 6) Capacidad local \(K_{\text{local}}\)

Cuando un Marcador está en una celda \(p\) y evalúa crear una baliza, define el conjunto de balizas “locales” como:

\[
B_{\text{local}}(p)=\{b_k\in B\;|\; d_\infty(c_k,p)\le r_b\}
\]

La capacidad local impone:

\[
|B_{\text{local}}(p)| \le K_{\text{local}}
\]

---

## 7) Regla del Marcador: asignar vs crear baliza

Sea \(i\) el objeto seleccionado (primer objeto sin etiqueta en orden de lectura dentro de \(N_1\)).

### 7.1 Balizas visibles

El Marcador mira balizas en rango del objeto:

\[
B_{\text{vis}}(i)=\{b_k\in B\;|\; d_\infty(c_k,pos(i))\le r_b\}
\]

### 7.2 Distancia al prototipo

Si \(B_{\text{vis}}(i)\neq \emptyset\), calcula:

\[
k^* = \arg\min_{b_k\in B_{\text{vis}}(i)} \|\tilde{x}_i-\mu_k\|_2
\]

\[
d^*=\|\tilde{x}_i-\mu_{k^*}\|_2
\]

### 7.3 Decisión (política final, sin ambigüedades)

**Caso 1 — No hay balizas visibles** (\(B_{\text{vis}}(i)=\emptyset\))

Acción:
- crear nueva baliza \(b_{K+1}\) en \(pos(i)\),
- asignar \(\ell(i)=K+1\),
- inicializar \(\mu_{K+1}=\tilde{x}_i,\; n_{K+1}=1\).

**Caso 2 — Hay balizas visibles** (\(B_{\text{vis}}(i)\neq\emptyset\))

1) Calcular \(k^*\) y \(d^*\).

2) Decidir:
- Si \(d^*\le \tau\): asignar a \(k^*\) y actualizar \(\mu_{k^*}\).
- Si \(d^*>\tau\):
  - si \(|B_{\text{vis}}(i)|<K_{\text{local}}\): crear nueva baliza y asignar,
  - si \(|B_{\text{vis}}(i)|=K_{\text{local}}\): asignar a \(k^*\) (fallback por similitud).

---

## 8) Regla del Transportador: recoger y depositar

### 8.1 Selección de objeto a recoger

En su vecindario \(N_1\), el Transportador elige el primer objeto con \(\ell(i)\neq 0\). Si no hay, ejecuta random walk.

### 8.2 Encontrar la baliza del tipo \(k=\ell(i)\)

En el modelo hay una baliza por etiqueta, por tanto existe \(b_k\) si \(\ell(i)\neq 0\).

El Transportador en cada paso:
- si detecta \(b_k\) con \(d_\infty(c_k,pos(T))\le r_b\), avanza greedy hacia \(c_k\);
- si no, random walk hasta detectarla.

### 8.3 Movimiento greedy (8 direcciones)

Elige \(q\in N_1(pos(T))\) que minimice:

\[
d_\infty(q,c_k)
\]

Desempate: primer candidato por orden.

### 8.4 Depósito: celda libre más cercana a la baliza

Busca el mínimo \(r\ge 1\) tal que exista una celda libre \(q\in N_r(c_k)\).  
Elige la primera celda libre por orden (o la más cercana según \(d_\infty\), equivalente dentro de \(N_r\)).

Si al llegar está ocupada, reinicia la búsqueda radial.

---

## 9) Scheduling y parada

### Orden de actualización

Asíncrono con orden estricto:
- en cada iteración \(t=1,\dots,T\),
- se actualizan Marcadores y Transportadores en el orden fijo de una lista.

### Parada

\[
t = 1,\dots,T \quad \text{(T fijo grande)}
\]

---

## 10) Seeds y reproducibilidad

Defina semillas separadas para:
- placement inicial,
- movimiento aleatorio,
- (si aplica) muestreo de \(\tau\) (en el caso exacto no es necesario, pero se deja parametrizado).

---

# PSEUDOCÓDIGO

## Algorithm 1 — OIC-Grid (pseudocódigo extendido)

```text
Algorithm 1: Opsonization-Inspired Clustering on a 2D Operational Grid (OIC-Grid)

Inputs:
  X = {x_i}_{i=1..N},  x_i ∈ R^d
  λ = 4                               // grid scaling
  r_obj = 1                           // object sensing radius (Moore 3×3)
  r_b                                 // beacon listening radius (hyperparameter)
  K_local                             // max visible beacons allowed locally (hyperparameter)
  T_iter                              // fixed number of iterations
  nM, nT                              // #Markers, #Transporters
  tau_sample_ratio ρ ∈ (0,1]          // optional: sampling ratio for τ estimation
  seed_place, seed_walk, seed_tau     // random seeds (placement, walk, tau-sampling)

Preprocessing:
  1) Z-score normalize all features:
       x̃_i ← zscore(x_i)  for i=1..N

Novelty threshold:
  2) P ← {(i,j) : 1 ≤ i < j ≤ N}                         // all index pairs
  3) If ρ < 1 then
       P ← sample_uniform_without_replacement(P, floor(ρ·|P|), seed_tau)
     end if
  4) D ← { ||x̃_i - x̃_j||_2 : (i,j) ∈ P }               // pairwise distances (exact if ρ=1)
  5) τ ← percentile_15(D)

Operational grid:
  6) |G| ← λ · N
  7) (W, H) ← grid_dimensions(|G|)                       // W=ceil(sqrt(|G|)), H=ceil(|G|/W)
  8) Initialize empty occupancy grid Occ[W,H]            // stores object id or empty
  9) For each object i=1..N:
       pos(i) ← sample_free_cell(Occ, seed_place)
       Occ[pos(i)] ← i

State variables:
 10) Initialize labels ℓ(i) ← 0  for all i               // 0 = unlabelled
 11) Initialize discarded(i) ← False for all i           // True => removed from dynamics (finalized)
 12) Initialize beacons B ← ∅, K ← 0                     // K = current number of beacons

Agents:
 13) Initialize Markers M_1..M_nM at random grid cells
 14) Initialize Transporters T_1..T_nT at random grid cells
 15) Each transporter has state carry ← None

Definitions:
  - N_r(p): set of cells within Chebyshev radius r from cell p
  - d∞(p,q): Chebyshev distance between cells p and q
  - ordered_list_of_objects_in_cells(S): objects in S sorted by deterministic reading order
  - greedy_step_minimize_dinf(p, target): one 8-dir step from p reducing d∞(·, target) the most
  - random_walk_step_8dir(p): one 8-dir random step from p
  - nearest_free_cell_by_increasing_radius(center=c, Occ): first free cell around c by increasing Chebyshev radius

Main loop (asynchronous, strict agent order):
  for t = 1..T_iter do
    for each agent a in [M_1..M_nM, T_1..T_nT] in fixed list order do

      // Marker agents (decision)
      if a is a Marker then
        S_cells ← N_r(pos(a), r_obj)
        C ← ordered_list_of_objects_in_cells(S_cells)
             filtered by (ℓ(i)=0 AND discarded(i)=False)

        if C is empty then
          pos(a) ← random_walk_step_8dir(pos(a), seed_walk)
          continue
        end if

        i ← first(C)                     // deterministic selection
        p ← pos(i)

        B_vis ← { b_k ∈ B : d∞(c_k, p) ≤ r_b }

        if B_vis is empty then
          K ← K + 1
          create beacon b_K:
            c_K ← p
            μ_K ← x̃_i
            n_K ← 1
          B ← B ∪ {b_K}
          ℓ(i) ← K

        else
          k* ← argmin_{b_k ∈ B_vis} || x̃_i - μ_k ||_2
          d* ← || x̃_i - μ_{k*} ||_2

          if d* ≤ τ then
            ℓ(i) ← k*
            n_{k*} ← n_{k*} + 1
            μ_{k*} ← μ_{k*} + (1/n_{k*}) · (x̃_i - μ_{k*})

          else
            if |B_vis| < K_local then
              K ← K + 1
              create beacon b_K:
                c_K ← p
                μ_K ← x̃_i
                n_K ← 1
              B ← B ∪ {b_K}
              ℓ(i) ← K
            else
              ℓ(i) ← k*
              n_{k*} ← n_{k*} + 1
              μ_{k*} ← μ_{k*} + (1/n_{k*}) · (x̃_i - μ_{k*})
            end if
          end if
        end if

        pos(a) ← random_walk_step_8dir(pos(a), seed_walk)

      // Transporter agents (action)
      else if a is a Transporter then

        if a.carry is None then
          S_cells ← N_r(pos(a), r_obj)
          C ← ordered_list_of_objects_in_cells(S_cells)
               filtered by (ℓ(i)≠0 AND discarded(i)=False)

          if C is empty then
            pos(a) ← random_walk_step_8dir(pos(a), seed_walk)
            continue
          end if

          i ← first(C)
          a.carry ← i
          Occ[pos(i)] ← empty             // remove object from grid (now carried)

        else
          i ← a.carry
          k ← ℓ(i)                        // invariant: ℓ(i)≠0 ⇒ beacon b_k exists

          // If beacon is not in listening range, keep exploring
          if d∞(pos(a), c_k) > r_b then
            pos(a) ← random_walk_step_8dir(pos(a), seed_walk)
            continue
          end if

          // Compute final target cell around beacon, then move toward it
          q ← nearest_free_cell_by_increasing_radius(center=c_k, Occ)
          if q does not exist then
            pos(a) ← random_walk_step_8dir(pos(a), seed_walk)
            continue
          end if

          pos(a) ← greedy_step_minimize_dinf(pos(a), q)

          if pos(a) == q then
            Occ[q] ← i
            pos(i) ← q
            a.carry ← None
            discarded(i) ← True            // finalize object (no further moves)
          end if
        end if
      end if

    end for
  end for

Output:
  - Labels ℓ(i) for all objects
  - Beacons B = { (c_k, μ_k, n_k) } for k=1..K
  - Final object positions pos(i)
  - Discarded flags discarded(i)

```

# SECCIÓN EXPERIMENTAL — PROTOCOLO, MÉTRICAS Y BASELINES

Esta sección describe **de forma reproducible y trazable** el diseño experimental utilizado para evaluar OIC-Grid y compararlo con un baseline estándar (*k-means*). El objetivo no es optimizar resultados, sino **validar empíricamente el comportamiento del algoritmo bajo métricas de clustering aceptadas**.

---

## 1) Objetivo experimental

Evaluar el rendimiento de **OIC-Grid** en tareas de clustering no supervisado y compararlo con **k-means** bajo un protocolo común, utilizando:

- datasets tabulares clásicos,
- normalización equivalente,
- métricas externas e internas estándar,
- múltiples ejecuciones con semillas distintas.

---

## 2) Datasets utilizados

Se emplean dos datasets ampliamente aceptados en la literatura de clustering:

### 2.1 Iris
- \(N = 150\) instancias
- \(d = 4\) características continuas
- \(K_{\text{real}} = 3\) clases
- Dataset balanceado y bien separado geométricamente

### 2.2 Wine
- \(N = 178\) instancias
- \(d = 13\) características continuas
- \(K_{\text{real}} = 3\) clases
- Dataset de mayor dimensionalidad y dispersión

Ambos datasets permiten:
- evaluación con métricas externas (etiquetas reales conocidas),
- comparación directa con k-means fijando \(k = K_{\text{real}}\).

---

## 3) Preprocesamiento común

Para **todos los métodos** (OIC-Grid y k-means):

- normalización *z-score* por característica:
\[
\tilde{x}_{i,f}=\frac{x_{i,f}-\mu_f}{\sigma_f}
\]

Esto garantiza que:
- la distancia euclídea sea comparable entre métodos,
- no existan sesgos por escala de variables.

---

## 4) Métricas de evaluación

### 4.1 Métricas externas (principales)

Se evalúan usando las etiquetas reales \(y_i\):

- **Adjusted Rand Index (ARI)**  
  Mide concordancia entre particiones ajustada por azar.  
  \[
  \text{ARI} \in [-1,1], \quad 1 = clustering perfecto
  \]

- **Normalized Mutual Information (NMI)**  
  Mide información compartida entre etiquetas reales y predichas.  
  \[
  \text{NMI} \in [0,1]
  \]

Estas métricas **no se utilizan durante el algoritmo**, solo para evaluación final.

---

### 4.2 Métricas internas (de apoyo)

Se calculan sin usar etiquetas reales:

- **Silhouette Score**
\[
s(i)=\frac{b(i)-a(i)}{\max(a(i),b(i))}
\]
donde \(a(i)\) es la distancia media intra-clúster y \(b(i)\) la mínima inter-clúster.

- **Davies–Bouldin Index**
\[
DB=\frac{1}{K}\sum_{k=1}^K \max_{j\neq k} \frac{\sigma_k+\sigma_j}{d(\mu_k,\mu_j)}
\]
Valores menores indican mejor separación.

Estas métricas se usan como **apoyo interpretativo**, no como criterio de optimización.

---

## 5) Protocolo experimental para OIC-Grid

### 5.1 Parámetros explorados

Se realiza un **grid search en dos niveles** sobre los hiperparámetros clave:

- \(r_b\): radio de escucha de balizas  
- \(K_{\text{local}}\): capacidad local máxima  
- \(n_M\): número de Marcadores  
- \(n_T\): número de Transportadores  
- \(T\): número fijo de iteraciones  

Otros parámetros (λ, r_obj) permanecen constantes.

---

### 5.2 Grid search en dos niveles

**Nivel 1 — Exploración amplia**
- pocas semillas por configuración,
- objetivo: filtrar configuraciones inviables o claramente malas.

**Nivel 2 — Evaluación robusta**
- múltiples ejecuciones por configuración finalista,
- agregación estadística de resultados.

Para cada configuración se reporta:
\[
\text{media} \pm \text{desviación típica}
\]

---

### 5.3 Seeds y repetibilidad

Para cada ejecución se fijan semillas independientes para:

- placement inicial de objetos en el grid,
- movimientos aleatorios (random walk),
- (si aplica) muestreo para el cálculo de \(\tau\).

Esto permite:
- evaluar estabilidad del algoritmo,
- evitar resultados dependientes de una única inicialización.

---

## 6) Protocolo experimental para k-means (baseline)

El baseline se implementa con:

- normalización *z-score* idéntica,
- número de clústeres fijado a \(k = K_{\text{real}}\),
- inicialización *k-means++*,
- múltiples inicializaciones internas (n_init).

Se calculan las **mismas métricas finales** que para OIC-Grid.

No se reporta desviación típica en el paper cuando el resultado es determinista bajo la configuración empleada.

---

## 7) Agregación y reporting de resultados

Para cada dataset y método se reportan:

- ARI (media ± std para OIC-Grid),
- NMI (media ± std para OIC-Grid),
- Silhouette (media ± std),
- Davies–Bouldin (media ± std).

Los resultados se exportan a CSV y se resumen en tablas comparativas utilizadas en el paper.

---

## 8) Interpretación experimental

El diseño experimental permite concluir que:

- *k-means* supera consistentemente a OIC-Grid en datasets tabulares estáticos,
- este resultado es coherente con la naturaleza distribuida y no optimizadora del algoritmo propuesto,
- OIC-Grid no está diseñado para maximizar directamente métricas globales, sino para estudiar **clasificación emergente bajo reglas locales**.

---

## 9) Relación con el notebook

El notebook experimental implementa exactamente este protocolo e incluye:

- ejecución automática del grid search,
- agregación estadística por configuración,
- generación de tablas finales,
- comparación directa con k-means.

El notebook constituye la **fuente ejecutable** de todos los resultados reportados en el paper y en este README.
