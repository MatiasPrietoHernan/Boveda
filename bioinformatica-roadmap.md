---
type: Note
status: Active
tags: [bioinformatica, sistemas, biologia, autodidacta, roadmap, papers]
---

# 🧬 Bioinformática y Biología de Sistemas — Roadmap Autodidacta

*Porque a los 20 años y siendo CTO no necesitás un título, necesitás código.*

---

## ¿Por qué biología de sistemas?

Es el **punto justo** entre lo que ya sabés y lo nuevo:

| Ya sabés | Aplicación en bioinfo |
|----------|----------------------|
| 🐍 Python + FastAPI | Servicios de análisis genómico |
| 📊 Data pipelines | Procesamiento de secuencias, OMICs |
| 🧠 LLMs + RAG | Extracción de conocimiento de papers biológicos |
| 🔗 Grafos + Qdrant | Redes de interacción de proteínas (PPI) |
| 🏗️ Microservicios | Desacoplar análisis bioinformáticos |

Biología de sistemas no es solo biología — es **modelar sistemas complejos con datos**, y eso es justo lo que ya hacés en waiprop.

---

## 📚 Los Papers Fundacionales (del más básico al más avanzado)

### 🟢 Nivel 1 — Panorama general (leer primero)

| Paper | Por qué es importante |
|-------|---------------------|
| **Kitano — "Systems Biology: A Brief Overview"** (Science, 2002) | El manifiesto de la disciplina. Cortito, claro. |
| **Ideker et al. — "A new approach to decoding life"** (Annu Rev, 2001) | Visión fundacional: cómo pensar la biología como sistema |
| **Noble — "The aims of systems biology"** (BioSystems, 2008) | Filosófico, te da el "por qué" |

### 🟡 Nivel 2 — Redes y modelado

| Paper | Por qué |
|-------|---------|
| **Barabási & Oltvai — "Network biology"** (Nat Rev Genet, 2004) | La biblia de redes biológicas. **Obligatorio.** |
| **Strogatz — "Exploring complex networks"** (Nature, 2001) | Bases matemáticas de redes (ya tenés ML, esto es pan comido) |
| **Bonneau — "The Inferelator"** (2006) | Conecta ML con redes de genes. **Acá podés aplicar lo tuyo.** |

### 🔴 Nivel 3 — Profundización

| Paper | Por qué |
|-------|---------|
| **Kauffman — "The Origins of Order"** (libro, 1993) | Auto-organización biológica. Pesado pero crucial. |
| **Bordbar et al. — "Constraint-based models"** (Nat Rev, 2014) | Modelos metabólicos (FBA, COBRA) |
| **Zomaya — "Bioinformatics and systems biology"** (2009) | Práctico, un puente entre ambas |

### 📘 El libro que tenés que tener

> **Alon — "An Introduction to Systems Biology: Design Principles of Biological Circuits"** (2007)
> No es un paper, es **el** libro. Arrancá por acá. Disponible online.

---

## 🛠️ Stack de herramientas

| Biblioteca | Para qué |
|------------|----------|
| **Biopython** | La navaja suiza: secuencias, PDB, KEGG, NCBI |
| **cobra** | Modelado metabólico (Flux Balance Analysis) |
| **networkx** | Grafos biológicos (PPI, metabolic networks) — ya la conocés |
| **pydeseq2** | Expresión diferencial (el estándar DESeq2 en Python) |
| **gseapy** | Enriquecimiento de pathways |
| **torch-geometric** | GNNs en grafos biológicos — **acá está el oro** |
| **BioServices** | Acceso programático a KEGG, UniProt, etc. |

---

## 🗺️ Roadmap 3 meses (sin dejar Waichatt)

### Semana 1-2 — Inmersión leve (~30 min/día)
- [ ] Leer Kitano (2002) — 20 min
- [ ] Leer Ideker (2001) — 20 min
- [ ] Instalar Python bio stack: `pip install biopython cobra networkx pydeseq2`
- [ ] Descargar dataset GEO chico (GSE25191 — yeast stress response)
- [ ] Familiarizarse con formatos: FASTA, FASTQ, BAM (solo saber qué son)

### Semana 3-4 — Primer proyecto concreto
- [ ] Construir **red de co-expresión génica** con `networkx` (correlación Pearson/Spearman)
- [ ] Aplicar detección de comunidades (Louvain/Leiden)
- [ ] Comparar con pathways conocidos de KEGG
- [ ] **Resultado:** un grafo de genes co-expresados que podés visualizar

### Mes 2 — Subir de nivel
- [ ] Implementar expresión diferencial con `pydeseq2`
- [ ] Hacer enriquecimiento GO/KEGG con `gseapy`
- [ ] Construir un modelo metabólico simple con `cobra` (E. coli core)
- [ ] **Resultado:** pipeline completo: raw data → genes diferenciales → pathways

### Mes 3 — El golpe maestro (combiná con IA)
- [ ] Construir **GNN** sobre red PPI (protein-protein interaction)
- [ ] Usar STRING DB + `torch-geometric` para predecir interacciones
- [ ] Crear un **asistente agéntico de biología de sistemas** con tu arquitectura actual
- [ ] **Resultado:** un microservicio FastAPI que responde preguntas biológicas con datos reales

---

## 🧪 Proyectos prácticos inmediatos

### 1. 🕸️ "Gene Prioritizer" — Priorización de genes por fenotipo
```
Input: lista de genes asociados a una enfermedad
Proceso: STRING PPI + random walk con restart
Output: ranking de genes candidatos
Deploy: FastAPI + Qdrant (lo mismo que waiprop)
```

### 2. 🔬 "Omics + LLM" — Co-expresión anotada con IA
```
Input: dataset GEO de expresión génica
Proceso: red de co-expresión → clustering → LLM anota cada cluster
Output: clústeres con descripciones funcionales en lenguaje natural
```

### 3. 🤖 "Agente biológico" — Asistente de biología de sistemas
```
Input pregunta: "¿Qué rutas metabólicas se alteran en diabetes?"
Proceso: agent query → KEGG + BioGRID + STRING → enrichment analysis
Output: respuesta en español con datos reales, sin alucinar
Esto es literalmente La Juga 🧉 aplicada a biología
```

---

## 🌐 Datasets públicos para arrancar

| Dataset | Qué tiene | Link |
|---------|-----------|------|
| **NCBI GEO** | 5M+ muestras de expresión génica | https://www.ncbi.nlm.nih.gov/geo/ |
| **STRING** | 24M+ proteínas, interacciones | https://string-db.org/ |
| **BioGRID** | 2.5M+ interacciones genéticas | https://thebiogrid.org/ |
| **KEGG** | Mapas de pathways (acceso gratis académico) | https://www.kegg.jp/ |
| **TCGA** | Multi-ómicas de cáncer | https://portal.gdc.cancer.gov/ |
| **GTEx** | Expresión por tejido | https://gtexportal.org/ |

---

## 🎓 Cursos online gratuitos

| Curso | Plataforma | Tiempo |
|-------|------------|--------|
| **MIT 7.91J — Foundations of Systems Biology** | MIT OCW | ~30hs |
| **Systems Biology and Biotechnology** | Coursera (Icahn) | ~20hs |
| **Genomic Data Science** | Coursera (Johns Hopkins) | ~25hs |
| **Machine Learning for Healthcare** | MIT OCW | ~40hs (toca ómica) |

---

## 💡 El consejo de Fangy

> No estudies bioinformática como un biólogo. **Estudiala como un ingeniero de sistemas.**

Tu ventaja no es saber biología — es saber **modelar, automatizar y escalar**. La biología de sistemas moderna es 70% data engineering, 20% ML y 10% biología. Ese 70% ya lo tenés.

Arrancá por el proyecto 1 (Gene Prioritizer). En una semana tenés un microservicio corriendo. Después le metés los papers de fondo.

**Cuando tengas dudas, las documentamos en la Bóveda.** 🧬🐺
