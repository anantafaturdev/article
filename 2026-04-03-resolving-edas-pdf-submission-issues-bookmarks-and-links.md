---
title: "Resolving EDAS PDF Submission Issues: Bookmarks and Links"
slug: resolving-edas-pdf-submission-issues-bookmarks-and-links
date_published: 2026-04-03T10:58:55.000Z
date_updated: 2026-04-03T10:59:53.000Z
excerpt: Submission to EDAS failed due to Zotero citation links and hidden PDF bookmarks. The solution was to unlink citations in a copy of the Google Docs file and remove all bookmarks using qpdf without altering the layout.
---

When uploading my final manuscript to EDAS, I encountered the following errors:

- **pdf links (URLs) are not allowed (**[**FAQ 221**](https://edas.info/faq221)**)**
- **pdf Bookmarks are not allowed (**[**FAQ 115**](https://edas.info/faq115)**)**

These errors prevented submission. Although they may seem like small issues, in practice they were not trivial and took a long time to debug, especially because I was using Google Docs instead of Microsoft Word. Honestly, even though the issue is “simple,” I spent around 2 hours solving it because I don’t fully understand how Google Docs or Word handles bookmarks and links; I’m just a DevOps engineer.

---

## Issue 1: Links (URLs) from Zotero

I used Zotero as my citation tool, which by default creates links between in-text citations, the references section, and the Zotero application. While these links are helpful during writing, EDAS detects them as URLs and rejects the PDF.

The solution is relatively straightforward. First, create a copy of the Google Docs file and open the copied version. Then use Zotero → Unlink Citations to remove all links.

It is important to always create a copy before unlinking, since the process is irreversible and citations cannot be relinked automatically afterward. For any further revisions, edits should be made in the main or master document, followed by creating a new copy and unlinking citations again before submission.

---

## Issue 2: Bookmarks (Unexpected and Hidden)

This part was more confusing. In Google Docs, a bookmark is a feature that acts as a pointer to a specific location in the document. However, during writing, I did not intentionally use bookmarks at all. 

Despite that, EDAS reported that the PDF contained bookmarks. At this point, I assumed it was a simple issue and tried removing bookmarks using PDF tools.

---

### Attempt 1: BentoPDF (Self-hosted)

Since the document was confidential, I could only use self-hosted PDF tools and could not rely on public services. I first tried BentoPDF, exploring features such as removing annotations, securing the PDF, and editing bookmarks. Despite these efforts, EDAS still rejected the file.

### Attempt 2: StirlingPDF (Self-hosted)

Because BentoPDF did not work, I tried another self-hosted tool, StirlingPDF. I repeated the same steps, but the result was unchanged: EDAS continued to detect bookmarks. At this stage, I needed to confirm whether bookmarks were actually present in the PDF.

### Attempt 3: Adobe Acrobat

During my research, I found a discussion on the Adobe forum about removing bookmarks and decided to try Adobe Acrobat. After opening the file, I made two findings: the old PDF file contained bookmarks, but the new PDF file processed through BentoPDF did not show any bookmarks. I also tried to remove bookmarks in Acrobat as suggested by the forum. Despite this, EDAS still rejected the file. This was confusing because Acrobat could read the bookmark list in my old PDF but showed none in the new PDF, yet EDAS continued to detect bookmarks. It made me realize there must be hidden bookmarks that Acrobat does not show.

### Attempt 4: Microsoft Word (Hidden Bookmarks)

To investigate further, I downloaded the document as a `.docx` and opened it in Microsoft Word. By checking Insert → Bookmark → Hidden bookmarks, I found multiple hidden bookmarks. It was possible to delete them manually, but the layout changed and the page count increased from 6 to 7 pages. Since this was a camera-ready paper, I did not want to risk altering the layout.

### Verification with `pdftk`

To answer the question: *does the PDF really contain bookmarks?*

I used `pdftk your.pdf dump_data | grep Bookmark`. The result:

    BookmarkTitle: I. INTRODUCTION
    BookmarkTitle: II. EXPERIMENTAL METHODOLOGY
    ...
    

This confirmed that bookmarks were indeed present in the PDF, even though some tools did not display them.

### Final Solution: Removing Bookmarks with `qpdf`

The goal was to remove bookmarks from the PDF metadata without altering the layout. I first installed `qpdf` using `sudo apt install qpdf`. Then, I rebuilt the PDF without bookmarks by running `qpdf --empty --pages your.pdf 1-z – no_bookmarks.pdf`.

To verify that all bookmarks were removed, I used `pdftk no_bookmarks.pdf dump_data | grep Bookmark`, which returned `Warning: no info dictionary found`. After this step, the PDF was finally compliant with EDAS requirements.

---

## Final Result

After successfully removing all links and hidden bookmarks, I uploaded the cleaned PDF to EDAS. The paper was accepted, but two warnings appeared: `Not certified by PDF eXpress”` and `“PDF version 1.3 (recommended 1.4+)”`. 

These warnings did not block the submission, and the committee indicated that authors do not need to use PDF eXpress themselves, so I considered them safe to ignore for now.

> Honestly i feel really tired, it took me around 2 hours to solve this because of my limited understanding and i couldn’t even share the story with `adalahpokoknya` since we’re in our silent phase, hiks 😭🥀
