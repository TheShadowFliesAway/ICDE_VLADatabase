# IEEEtran Formatting Guide for ICDE Drafts

Source: `IEEEtran_HOWTO.pdf`, "How to Use the IEEEtran LaTeX Class", Michael Shell, IEEEtran v1.8b, 2015.

This note is a project-local writing and formatting checklist. It summarizes the IEEEtran rules that matter when drafting an ICDE-style conference paper in this repository. Use the official ICDE author kit as the final authority when it differs from this guide.

## Working Baseline

- Use the official ICDE template when available.
- The current project template uses:

```tex
\documentclass[conference]{IEEEtran}
```

- Do not change margins, fonts, paper size, section spacing, column layout, or caption styles by hand unless the official ICDE instructions explicitly require it.
- Prefer `latexmk -pdf main.tex` for normal builds. Use `latexmk -xelatex main.tex` only if the template or fonts require XeLaTeX.
- Keep the generated PDF on US letter paper unless the official call asks for another size.

## Class Options

Use IEEEtran class options instead of manual formatting changes.

- `conference`: normal IEEE conference paper mode. This is the expected starting point for this repository.
- `journal`: journal mode, not for ICDE conference submissions.
- `technote`: correspondence/brief/technote mode, usually with `9pt`; not for ICDE.
- `peerreview` / `peerreviewca`: review-specific journal modes; not typical for ICDE conference submissions.
- `draft`, `draftcls`, `draftclsnofoot`: draft modes. Use `draftcls` or `draftclsnofoot` if figures should still render.
- `onecolumn`: useful for drafts only. IEEE camera-ready papers are two-column.
- `letterpaper` / `a4paper`: IEEE primarily uses US letter. Use `a4paper` only if the venue explicitly requires it.
- `compsoc`: only for IEEE Computer Society formats that explicitly require it. Many IEEE Computer Society conferences still use traditional conference mode, so do not enable it by guesswork.
- `comsoc` and `transmag`: special society modes; not relevant to ICDE unless official instructions say so.

## Preamble Rules

- Let IEEEtran control fonts and spacing.
- Do not load packages that replace the IEEE fonts, such as old Times/font hacks, unless the venue requires them.
- Do not use `geometry` or similar packages to alter margins.
- Use `graphicx` for figures.
- Use `cite` for IEEE-style numeric citations.
- Use `url` for URLs and email addresses.
- Use `amsmath` when equations need richer alignment, but if multiline equations may break across pages, add:

```tex
\usepackage{amsmath}
\interdisplaylinepenalty=2500
```

- Avoid loading `IEEEtrantools` with `IEEEtran`; those tools are already provided by the class.

## Title And Authors

Declare title, authors, abstract, keywords, and then call `\maketitle`.

```tex
\title{Paper Title}

\author{\IEEEauthorblockN{Anonymous Authors}
\IEEEauthorblockA{Paper under submission}}

\maketitle
```

Rules:

- Capitalize titles in IEEE style: major words capitalized; short function words such as "a", "an", "and", "as", "at", "by", "for", "in", "of", "on", "or", "the", "to", and "up" are usually lowercase unless first or last.
- Do not put math or special symbols in the title.
- Use `\\` in the title only when needed to balance title lines.
- In conference mode, use `\IEEEauthorblockN{}` for names and `\IEEEauthorblockA{}` for affiliations.
- Use `\and` to separate affiliation columns when there are three or fewer affiliation blocks.
- For many authors or wide affiliation text, use the long author format with `\IEEEauthorrefmark{}`.
- Conference papers do not use running headers; `\markboth{}{}` has no effect in conference mode.
- Do not add publication ID marks with `\IEEEpubid{}` in camera-ready conference papers.

## Abstract And Keywords

Use the standard IEEE environments:

```tex
\begin{abstract}
...
\end{abstract}

\begin{IEEEkeywords}
...
\end{IEEEkeywords}
```

Rules:

- Avoid math, special symbols, and citations in the abstract when possible.
- Avoid math and special symbols in keywords.
- Keywords should be concise and conventional for the field.
- In conference mode, the abstract and keywords appear after `\maketitle` and before the first section.

## Sections

Use normal LaTeX sectioning:

```tex
\section{Introduction}
\subsection{...}
\subsubsection{...}
```

Rules:

- Do not manually style section headings.
- Keep nesting shallow. Deep `\paragraph`-level structure is usually not appropriate for conference papers.
- For ICDE drafts, use section titles that support scanning: `Introduction`, `Motivation`, `Problem Definition`, `Design`, `Implementation`, `Evaluation`, `Related Work`, `Conclusion`.

## Citations

Use IEEE numeric citation style.

```tex
\usepackage{cite}
...
Prior work shows ...~\cite{key1,key2,key3}.
```

Rules:

- Put adjacent citations in a single `\cite{...}` command so `cite.sty` can sort and compress them.
- Use nonbreaking space before citations in prose: `... method~\cite{key}.`
- Avoid citation notes with multiple references. If using a note, cite one reference:

```tex
\cite[Th. 7.1]{key}
```

- Do not manually format reference numbers.

## Equations

Use `equation` for numbered equations:

```tex
\begin{equation}
\label{eq:example}
x = \sum_{i=0}^{z} 2^i Q
\end{equation}
```

Use display math only when no equation number is needed.

Rules:

- Reference equations as `(\ref{eq:example})`, not "equation (1)".
- Make every equation fit the column width.
- Break long equations manually; do not shrink math fonts to force fit.
- Consider `amsmath` or IEEEtran's `IEEEeqnarray` for multiline equations.
- Avoid double-column equations unless truly necessary; IEEE rarely uses them and they require careful manual placement and numbering.
- For cases structures with separately referenceable branches, use the `cases` package's `numcases` or `subnumcases`, loaded after `amsmath`.

## Figures

Basic figure pattern:

```tex
\begin{figure}[!t]
\centering
\includegraphics[width=\columnwidth]{figures/example.pdf}
\caption{Simulation results for the network.}
\label{fig:example}
\end{figure}
```

Rules:

- Use `\centering`, not the `center` environment, inside floats.
- Put figure captions below figures.
- Put `\label{...}` after or inside `\caption{...}`.
- Prefer top placement: `[!t]`.
- Typical IEEE prose uses "Fig." for figure references. If using IEEEtran mode-dependent text, `\figurename` contains the correct form.
- Use vector PDF/EPS for line art, plots, diagrams, charts, and graphs.
- Bitmap formats are acceptable for photos or rendered images where vector form is not meaningful.
- Avoid low-resolution, pixelated, unembedded, or bitmap-font graphics.
- Use `draftcls` or `draftclsnofoot`, not plain `draft`, when drafting with figures visible.

## Subfigures

IEEEtran recommends `subfig` rather than obsolete `subfigure`. Use `caption=false` so IEEEtran keeps control of caption style.

```tex
\usepackage[caption=false,font=footnotesize]{subfig}
```

For Computer Society mode only, use the `compsoc` branch from the HOWTO. Do not enable that branch for ICDE unless the official template requires `compsoc`.

Rules:

- Most IEEE papers describe subfigures `(a)`, `(b)`, etc. in the main caption instead of giving each subfigure a full caption.
- Keep total subfigure width below `\textwidth` for `figure*` or below `\columnwidth` for single-column figures.
- Do not use `subcaption` unless the official template permits it, because it can override IEEEtran caption formatting.

## Algorithms

IEEE publications use only figure and table floats. The HOWTO explicitly warns against the floating environments from `algorithm.sty` and `algorithm2e.sty` because their captions are not IEEE-controlled.

Preferred pattern:

```tex
\begin{figure}[!t]
\centering
% algorithmic or custom pseudocode content here
\caption{Ingesting an observation.}
\label{fig:ingest-algorithm}
\end{figure}
```

Rules:

- Use `algorithmic` or `algorithmicx` for pseudocode layout if needed.
- Put the pseudocode inside a `figure` float, not an `algorithm` float, unless the ICDE template explicitly allows algorithm floats.
- If the current draft uses `\begin{algorithm}`, check the final PDF carefully and consider converting it to a `figure`.

## Tables

Basic table pattern:

```tex
\begin{table}[!t]
\renewcommand{\arraystretch}{1.3}
\caption{A Simple Example Table}
\label{tab:example}
\centering
\begin{tabular}{ll}
\hline
\bfseries Field & \bfseries Meaning\\
\hline
...
\hline
\end{tabular}
\end{table}
```

Rules:

- Put table captions above tables.
- Put `\label{...}` after or inside `\caption{...}`.
- Table captions are title-like and usually capitalized.
- Preserve units and math letters in captions with upright text when case changes could alter meaning, e.g. `{\upshape Hz}`.
- IEEE tables often use `\footnotesize` by default.
- Increase `\arraystretch` slightly when rows feel cramped.
- IEEE often uses open-sided tables, though closed-sided tables also appear.
- For table footnotes, prefer `threeparttable` or put notes at the end of the table rather than relying on page footnotes inside floats.

## Double-Column Floats

Use `figure*` and `table*` only when the content cannot fit a single column.

Rules:

- Double-column floats normally appear at the top of a later page.
- LaTeX usually cannot place double-column floats on the same page where they are defined.
- Avoid packages that put content across the middle of two columns; IEEE does not use that style.
- If bottom double-column floats are unavoidable, `dblfloatfix` is the recommended combined fix, but use it cautiously and inspect output.
- Check float order manually when mixing single-column and double-column floats.

## Lists

IEEEtran modifies `itemize`, `enumerate`, and `description` to follow IEEE list style.

Rules:

- Use normal list environments unless you need special label widths.
- For long enumerate labels, set the label width:

```tex
\begin{enumerate}[\IEEEsetlabelwidth{12)}]
...
\end{enumerate}
```

- For description lists with math symbols, set the longest label and use math label spacing:

```tex
\begin{description}[
  \IEEEsetlabelwidth{$\alpha\omega\pi\theta\mu$}
  \IEEEusemathlabelsep]
...
\end{description}
```

- Keep lists short and purposeful in conference papers. Long contribution lists are harder to read in two-column format.

## Theorems And Proofs

Declare theorem-like structures with `\newtheorem`.

```tex
\newtheorem{theorem}{Theorem}
```

Use IEEEtran's proof environment:

```tex
\begin{IEEEproof}
...
\end{IEEEproof}
```

Rules:

- Most IEEE papers use theorem numbering across the whole paper unless section-based numbering is needed.
- Use `\IEEEproof[Proof of Theorem~\ref{thm:...}]` when the proof label needs to be specific.

## Appendices

Use one appendix:

```tex
\appendix[Proof of the Main Lemma]
```

Use multiple appendices:

```tex
\appendices
\section{Proof of the First Result}
\section{Additional Experiments}
```

Rules:

- For a single appendix, do not write "Appendix A"; refer to "the Appendix".
- For multiple appendices, declare a `\section` before lower-level headings or labels.
- IEEEtran defaults to alphabetic appendix labels. Use `romanappendices` only when the venue asks for Roman-numbered appendices.
- ICDE conference submissions often have strict page limits; appendices may be disallowed or excluded from review. Follow the official instructions.

## Acknowledgments

Use an unnumbered section:

```tex
\section*{Acknowledgment}
```

Rules:

- IEEE Computer Society style often uses plural `Acknowledgments`.
- For anonymous review, remove or anonymize acknowledgments if required.
- For final camera-ready, restore funding and contributor acknowledgments according to the venue policy.

## Bibliography

Use IEEEtran BibTeX style:

```tex
\bibliographystyle{IEEEtran}
\bibliography{refs}
```

Rules:

- Do not hand-format references.
- Keep BibTeX entries complete: authors, title, venue, year, pages, DOI/URL when appropriate.
- Before source submission to an external party, IEEEtran recommends copying the generated `.bbl` into the document so the bibliography does not depend on external BibTeX files.
- For this repository during drafting, keep using `refs.bib` for maintainability.

## Last Page Column Balance

IEEE coarsely equalizes the two columns on the last page.

Rules:

- For camera-ready, inspect the last page manually.
- Manual fixes are preferred:

```tex
\newpage
```

or near the top of the first column of the last page:

```tex
\enlargethispage{-0.5in}
```

- Automatic packages such as `balance` and `flushend` can be unreliable, especially with figures and references. If used, inspect spacing carefully.

## PDF Output Quality

Before submission, check:

- Correct paper size.
- Correct margins.
- All fonts embedded and subsetted.
- No bitmap Type 3 fonts.
- Figures are crisp at high zoom.
- No overfull boxes in visible text.
- No unresolved references or citations.
- No warning-caused missing figures.

Useful commands:

```bash
latexmk -pdf conference_101719.tex
pdfinfo conference_101719.pdf
pdffonts conference_101719.pdf
```

## Common Mistakes To Avoid

- Placing `\label` before `\caption` in figures or tables.
- Manually changing margins, spacing, fonts, section headings, or column style.
- Loading old graphics packages instead of `graphicx`.
- Using bitmap graphics for plots, diagrams, or line art.
- Using unembedded fonts or Type 3 bitmap fonts.
- Letting equations run past the column width.
- Shrinking equation fonts instead of breaking equations properly.
- Hand-formatting references instead of using IEEEtran BibTeX style.
- Loading packages only because they were in an old template.
- Using `algorithm` or `algorithm2e` floating environments without checking whether the venue permits them.
- Using `subcaption` or caption packages that override IEEEtran caption formatting.

## Project-Specific Review Checklist

Before asking for paper-writing help or a formatting review, check:

- `\documentclass[conference]{IEEEtran}` is still intact unless official ICDE instructions say otherwise.
- Abstract has no citations unless absolutely necessary.
- Title has no math or special symbols.
- Figure captions are below figures; table captions are above tables.
- Every figure/table label appears after its caption.
- Citations are grouped and use `~\cite{...}`.
- Equations fit in one column or are deliberately broken.
- Algorithms are not using non-IEEE float styles unless explicitly allowed.
- Bibliography uses `\bibliographystyle{IEEEtran}`.
- Final PDF has embedded fonts and no visible layout warnings.
