# IR PDF Extraction
- [IR PDF Extraction](#ir-pdf-extraction)
- [Phase 1](#phase-1)
  - [Overview](#overview)
  - [Features](#features)
  - [Approach](#approach)
    - [Advantages of `pdfplumber`](#advantages-of-pdfplumber)
    - [Why Extract Document Structure Based on Font Size?](#why-extract-document-structure-based-on-font-size)
  - [Example Workflow](#example-workflow)
  - [Code Explanation](#code-explanation)
    - [1. **Paragraph Extraction**](#1-paragraph-extraction)
    - [2. **Font Beautification**](#2-font-beautification)
    - [3. **Recursive Grouping**](#3-recursive-grouping)
  - [Output Example](#output-example)
- [Phase 2](#phase-2)
  - [TODO](#todo)
- [Contributors](#contributors)



---

# Phase 1 


## Overview

In the first phase, the tool focuses on:
- **Extracting information** from a PDF file.
- **Structuring extracted data** in a hierarchical JSON format using font sizes and other attributes.
- **Automating the process** to adapt to varying text structures and styles.

---

## Features
1. **Automatic Text Extraction:** Parses the text from PDF documents using font size, font type, and position data.
2. **Font Analysis:** Identifies hierarchical headers (e.g., Header 1, Header 2) based on font sizes.
3. **Structured JSON Generation:** Groups the text hierarchically using a heuristic algorithm, creating a JSON output.

---
## Approach

In this implementation `pdfplumber` was used to extract text with atributes like font name and font size which were the base of our method for document structure extraction.

### Advantages of `pdfplumber`

1. **Comprehensive Text Extraction:**
   - Unlike many other PDF libraries, `pdfplumber` provides precise control over text extraction, including font sizes, font names, and character positions.
   - It handles complex layouts, including multi-column documents and irregular text formatting.

2. **Character-Level Details:**
   - It provides access to detailed character-level information, such as font size, font type, and position. This makes it ideal for hierarchical analysis, such as identifying headers and body text based on font size and formatting.

### Why Extract Document Structure Based on Font Size?

 1. **Replicates Visual Hierarchy**
- Font size is one of the primary visual indicators used to convey hierarchy in documents:
  - **Larger fonts** typically denote higher-level headings or titles.
  - **Smaller fonts** are often used for subheadings and body text.
- By analyzing font sizes, you can replicate this visual structure programmatically.

---

 2. **Preserves Logical Organization**
- Most documents, such as reports, books, or presentations, follow logical formatting:
  - Major sections and subsections have distinct font sizes.
  - Using font size ensures the logical flow of content is captured, even in text-heavy or complex layouts.



---
 3. **Handles Variability in Layouts**
- While document layouts may vary widely, font size remains a consistent feature for hierarchy. This makes it:
  - **Resilient** to changes in document design.
  - **Independent** of language or formatting nuances.

---
## Example Workflow
1. **Input:** First we open the PDF using `pdfplumber`.
2. **Extract metadata:**
   - Extract Lines using `extract_text_lines` method.
   - get font names & sizes from each line list of chars.
   - save the font size in a seperate list to find the most common font later.
3. **Process:** 
   - find the most used font in the document and create a set of fonts with their labels.
   - Recursively group text into a hierarchical structure:
     - 1. check the postion of the strating char to avoid saving footers and headers
     - 2. either returns or call the function again based on the changes in font size
     - 3. save the content with their corresponding labels 
4. **Output:** A structured JSON representation of the document.

---

## Code Explanation

### 1. **Paragraph Extraction**
The `extract_paragraphs` function:
- Uses `pdfplumber` to read the PDF file.
- Extracts text, font size, font type, and position for each paragraph.


### 2. **Font Beautification**
The `font_beautify` function:
- Identifies the most common font size as the "Body" text size.
- Maps other font sizes to hierarchical header levels (e.g., `Header 1`, `Header 2`).


### 3. **Recursive Grouping**
The `group_by_size_recursively` function:
- checks positions to avoid storing footers and headers.
```python
    if paragraph["position"] < 50:
          continue
```

- compare the current font size and previouse one, if the current one is bigger then we return to the caller and place the unprocessed object back to the list
```python
    if paragraph["size"] > current_size:
            result.append(lines)
            paragraphs.insert(0, paragraph) 
            return result
```
- in this next part of the code we try to merge non-body blocks of text(headers with the same font size) and add the merged and final text to a list of labels
  
```python
    if fonts[paragraph["size"]] != "Body":
          if current_obj is None:
                current_obj = {
                    "content": fonts[paragraph["size"]],
                    "text": paragraph["text"],
                    "size": paragraph["size"],
                    "position": paragraph["position"]
                }
          elif fonts[paragraph["size"]] == fonts[current_obj["size"]]:
              current_obj["text"] += " " + paragraph["text"]
          else:
            if not any(label["text"] == current_obj["text"] for label in labels):
               labels.append(current_obj)
            current_obj = None
```

- in case of no changes in the font size we simply save the currnt line:
```python
    if paragraph["size"] == current_size:
          if fonts[paragraph["size"]] == "Body":
            lines.append(paragraph["text"])
```
- but if the font size got smaller the we make a sublayer in our json and use the latest label in `labels`(or a Deadault value of "subLayer" for null-safety) as the sublayer key:
```python 
  else:
            if current_obj is not None:
              labels.append(current_obj)
              current_obj = None
            paragraphs.insert(0, paragraph)
            sub_layer = group_by_size_recursively(paragraphs, paragraph["size"], fonts, labels)


            if labels:
              label = labels.pop()
              label_text = label["text"]
            else:
              label_text =  "subLayer"


            result.append({
                label_text : sub_layer
            })
```
- With the final return of the function we Construct a nested JSON structure to represent the document hierarchy.




## Output Example
```json
"Diagnostic Features": [
                            [
                                "The essential feature of depressive disorder due to another medical condition is a promi-",
                                "nent and persistent period of depressed mood or markedly diminished interest or plea-",
                                "sure in all, or almost all, activities that predominates in the clinical picture (Criterion A)",
                                "and that is thought to be related to the direct physiological effects of another medical con-",
                                "dition (Criterion B). In determining whether the mood disturbance is due to a general",
                                "medical condition, the clinician must first establish the presence of a general medical con-",
                                "dition. Further, the clinician must establish that the mood disturbance is etiologically re-",
                                "lated to the general medical condition through a physiological mechanism. A careful and",
                                "comprehensive assessment of multiple factors is necessary to make this judgment. Al-",
                                "though there are no infallible guidelines for determining whether the relationship",
                                "between the mood disturbance and the general medical condition is etiological, several",
                                "considerations provide some guidance in this area. One consideration is the presence of a",
                                "temporal association between the onset, exacerbation, or remission of the general medical",
                                "condition and that of the mood disturbance. A second consideration is the presence of fea-",
                                "tures that are atypical of primary Mood Disorders (e.g., atypical age at onset or course or",
                                "absence of family history). Evidence from the literature that suggests that there can be a di-",
                                "rect association between the general medical condition in question and the development",
                                "of mood symptoms can provide a useful context in the assessment of a particular situation."
                            ]
                        ]
                    },
                    {
                        "Associated Features Supporting Diagnosis": [
                            [
                                "Etiology (i.e., a causal relationship to another medical condition based on best clinical ev-",
                                "idence) is the key variable in depressive disorder due to another medical condition. The",
                                "listing of the medical conditions that are said to be able to induce major depression is never",
                                "complete, and the clinician\u2019s best judgment is the essence of this diagnosis.",
                                "There are clear associations, as well as some neuroanatomical correlates, of depression",
                                "with stroke, Huntington\u2019s disease, Parkinson\u2019s disease, and traumatic brain injury. Among",
                                "the neuroendocrine conditions most closely associated with depression are Cushing\u2019s dis-",
                                "ease and hypothyroidism. There are numerous other conditions thought to be associated",
                                "with depression, such as multiple sclerosis. However, the literature\u2019s support for a causal",
                                "association is greater with some conditions, such as Parkinson\u2019s disease and Huntington\u2019s",
                                "disease, than with others, for which the differential diagnosis may be adjustment disorder,",
                                "with depressed mood."
                            ]
                        ]
                    },
                    {
                        "Development and Course": [
                            [
                                "Following stroke, the onset of depression appears to be very acute, occurring within 1 day",
                                "or a few days of the cerebrovascular accident (CVA) in the largest case series. However, in",
                                "some cases, onset of the depression is weeks to months following the CVA. In the largest",
                                "series, the duration of the major depressive episode following stroke was 9\u201311 months on",
                                "average. Similarly, in Huntington\u2019s disease the depressive state comes quite early in the",
                                "course of the illness. With Parkinson\u2019s disease and Huntington\u2019s disease, it often precedes",
                                "the major motor impairments and cognitive impairments associated with each condition.",
                                "This is more prominently the case for Huntington\u2019s disease, in which depression is con-",
                                "sidered to be the first neuropsychiatric symptom. There is some observational evidence",
                                "that depression is less common as the dementia of Huntington\u2019s disease progresses."
                            ]
                        ]
                    },
                    {
                        "Risk and Prognostic Factors": [
                            [
                                "The risk of acute onset of a major depressive disorder following a CVA (within 1 day to a",
                                "week of the event) appears to be strongly correlated with lesion location, with greatest risk",
                                "associated with left frontal strokes and least risk apparently associated with right frontal",
                                "lesions in those individuals who present within days of the stroke. The association with",
                                "frontal regions and laterality is not observed in depressive states that occur in the 2\u20136 months",
                                "following stroke."
                            ]
                        ]
                    },
                    {
                        "Gender-Related Diagnostic Issues": [
                            [
                                "Gender differences pertain to those associated with the medical condition (e.g., systemic",
                                "lupus erythematosus is more common in females; stroke is somewhat more common in",
                                "middle-age males compared with females)."
                            ]
                        ]
                    },
                    {
                        "Diagnostic Markers": [
                            [
                                "Diagnostic markers pertain to those associated with the medical condition (e.g., steroid",
                                "levels in blood or urine to help corroborate the diagnosis of Cushing\u2019s disease, which can",
                                "be associated with manic or depressive syndromes)."
                            ]
                        ]
                    },
                    {
                        "Suicide Risk": [
                            [
                                "There are no epidemiological studies that provide evidence to differentiate the risk of sui-",
                                "cide from a major depressive episode due to another medical condition compared with the",
                                "risk from a major depressive episode in general. There are case reports of suicides in",
                                "association with major depressive episodes associated with another medical condition.",
                                "There is a clear association between serious medical illnesses and suicide, particularly",
                                "shortly after onset or diagnosis of the illness. Thus, it would be prudent to assume that the",
                                "risk of suicide for major depressive episodes associated with medical conditions is not less",
                                "than that for other forms of major depressive episode, and might even be greater."
                            ]
                        ]
                    },
                    {
                        "Functional Consequences of Depressive Disorder Due to Another Medical Condition": [
                            [
                                "Functional consequences pertain to those associated with the medical condition. In gen-",
                                "eral, it is believed, but not established, that a major depressive episode induced by Cush-",
                                "ing\u2019s disease will not recur if the Cushing\u2019s disease is cured or arrested. However, it is also",
                                "suggested, but not established, that mood syndromes, including depressive and manic/",
                                "hypomanic ones, may be episodic (i.e., recurring) in some individuals with static brain in-",
                                "juries and other central nervous system diseases."
                            ]
                        ]
                    },
                    {
                        "Differential Diagnosis": [
                            [
                                "Depressive disorders not due to another medical condition. Determination of whether",
                                "a medical condition accompanying a depressive disorder is causing the disorder depends",
                                "on a) the absence of an episode(s) of depressive episodes prior to the onset of the medical",
                                "condition, b) the probability that the associated medical condition has a potential to pro-",
                                "mote or cause a depressive disorder, and c) a course of the depressive symptoms shortly",
                                "after the onset or worsening of the medical condition, especially if the depressive symp-",
                                "toms remit near the time that the medical disorder is effectively treated or remits.",
                                "Medication-induced depressive disorder. An important caveat is that some medical con-",
                                "ditions are treated with medications (e.g., steroids or alpha-interferon) that can induce depres-",
                                "sive or manic symptoms. In these cases, clinical judgment, based on all the evidence in hand, is",
                                "the best way to try to separate the most likely and/or the most important of two etiological fac-",
                                "tors (i.e., association with the medical condition vs. a substance-induced syndrome).",
                                "Adjustment disorders. It is important to differentiate a depressive episode from an ad-",
                                "justment disorder, as the onset of the medical condition is in itself a life stressor that could",
                                "bring on either an adjustment disorder or an episode of major depression. The major dif-",
                                "ferentiating elements are the pervasiveness the depressive picture and the number and",
                                "quality of the depressive symptoms that the patient reports or demonstrates on the mental",
                                "status examination. The differential diagnosis of the associated medical conditions is rel-",
                                "evant but largely beyond the scope of the present manual."
                            ]
                        ]
                    }
```



# Phase 2

# Document Retrieval and Ranking: A Comparative Approach

## Overview
This project focuses on developing and evaluating document retrieval and ranking systems using multiple approaches. The workflow includes the use of **TF-IDF**, **Boolean Query Models**, and **BERT** models for enhancing precision and recall. The performance is measured using **Precision@K (P@K)** and **Recall@K (R@K)** metrics.

---

## Methodology

### **1. TF-IDF-Based Analysis**
- **Objective:** Retrieve and rank documents based on their similarity to a query using the TF-IDF algorithm.
- **Process:** 
  - Extract text from PDF documents.
  - Build a TF-IDF vector representation of the documents.
  - Retrieve top-`K` documents based on cosine similarity scores.
- **Evaluation:**  
  - Metrics used: **Precision@K (P@K)** and **Recall@K (R@K)**.
  - Results demonstrated moderate precision but limited recall, indicating the need for improved methods.

---

### **2. Boolean Query Model**
- **Objective:** Enable users to query documents using Boolean operators (`AND`, `OR`, `NOT`).
- **Process:**  
  - Create an inverted index from document terms.
  - Represent documents as binary vectors.
  - Evaluate Boolean queries against the binary representation of documents.
- **Outcome:**  
  - Boolean queries provided an efficient way to filter documents, but their strict logic sometimes excluded relevant results.

---

### **3. Enhanced Model: BERT + TF-IDF Hybrid**
- **Objective:** Improve precision and recall by leveraging a **BERT-based retrieval model** in combination with **TF-IDF ranking**.
- **Process:**  
  - Use BERT embeddings for query and document vectors to retrieve a broader set of relevant documents.
  - Re-rank retrieved documents using the TF-IDF scoring model.
- **Evaluation:**  
  - Precision and recall showed significant improvement, as the hybrid model better captured semantic relationships between queries and documents.

---

## Results

### **Metrics Comparison**
| Model                 | Precision@K (P@5) | Recall@K (R@5) |
|-----------------------|-------------------|----------------|
| **TF-IDF**            | 0.82              | 0.24           |
| **Boolean Query**     | NA (Not ranked)   | Context-dependent |
| **BERT + TF-IDF**     | 0.93              | 0.48           |

### **Key Observations**
1. The **TF-IDF** model excels in identifying exact keyword matches but struggles with capturing semantic nuances.
2. The **Boolean Query Model** is fast and interpretable but lacks flexibility in ranking documents.
3. The **BERT + TF-IDF Hybrid Model** offers the best balance, significantly improving both precision and recall.

---

## Conclusion
The hybrid approach combining **BERT** for retrieval and **TF-IDF** for ranking provides the most effective solution for document retrieval. By addressing the limitations of individual models, the hybrid approach achieved superior performance on both precision and recall metrics. This demonstrates the importance of leveraging semantic understanding in modern document retrieval tasks.

---


# Contributors

This project was created and maintained by the following individuals:

- [Amirhosein Firoozi](https://github.com/AmirHoseinFRZ)
- [MohammadReza MirRashid](https://github.com/mmdreza00mirrashid)
- [Pedram Vahdati Rohani](https://github.com/Pedram5879)
