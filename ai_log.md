# AI log: Wave home assignment

A running record of how I used AI on this assignment, and every case where the AI or a source was wrong.
It feeds the mandatory "How I learned" slide.

## How AI is used

- **Claude (chat agent):** finding the brief, a first audio experiment on my own dryer recording, a research report on MikaGu/Wave and on DCASE Task 2, and drafting assumptions and clarifying questions.
- **Rule:** every claim gets checked against the brief or a primary source (DCASE result pages, the papers themselves) before it goes on a slide.

## Errors and corrections

| # | Date | What the AI said or did | What was wrong | How I caught it | Status |
|---|------|------------------------|----------------|-----------------|--------|
| 1 | 18 Nov | Before it had the brief, Claude assumed the task was about household appliances, based on my dryer recording, and briefed the research agent that way. | The brief is about **industrial machines in a factory**; the dryer was only my own test rig. The research report built on the wrong premise ("the dryer task is almost certainly a proxy"). | Reading the actual brief on 19 Nov. | Fixed. **Lesson:** give the AI the source document, not my summary of it. |
| 2 | 19 Nov | Claude summarised the brief as "the device sends a health score to the cloud". | The brief says a health score **per component** (לכל רכיב). That implies localising or diagnosing which part is failing, not only a machine-level flag. | Re-reading the brief line by line. | Fixed. |
| 3 | 19 Nov | Claude used "health score" in the clarifying questions without defining it. | The term is ambiguous: is it an anomaly score, a probability, or a 0–100 index? Questions must use precise terms. | My own review of the draft. | Fixed. Defined as an anomaly score per component per recording window. |
| 4 | 19 Nov | Claude's first Todoist plan used the interview day (26 Nov) as the deadline. | The brief requires submitting the deck **one day before** the meeting (25 Nov). | Reading the brief. | Fixed. |
| 5 | 19 Nov | The research report suggested asking the questions "on the 18 Nov call". | That call had already happened when the report was written. | Checking the dates. | Noted. |
| 6 | 18 Nov | The research report called Tal Kfir "Dr.", and listed Dorad and Dalia power stations as customers. | LinkedIn lists his PhD as in progress (2024–present). The customers are self-reported on the company site and not independently confirmed. | Caveats in the report itself, plus checking LinkedIn. | Don't state either in the interview. |
| 7 | 18 Nov | The research report cites 2026 arXiv preprints (2606.19269 on backend choice for BEATs, 2607.04526 on BEATs+kNN) with specific numbers. | They are not peer-reviewed, and the numbers are not verified yet. | Report caveat. | **Open:** read the papers before citing them. |
| 8 | 18 Nov | Hypothesis from the analysis and the report: the 100 and 300 Hz dryer peaks are 50 Hz mains harmonics, and the 80 Hz peak is mechanical. | Unverified. Phone microphones are unreliable below about 100 Hz. | Not yet checked. | **Open:** test with a longer recording and a better microphone. |
| 9 | 19 Nov | Claude said a CPU-only edge device can't run BEATs, so neural networks would need a GPU. | Overstated. With a 10-second clip every couple of minutes, BEATs (~90M parameters) runs fine on a Raspberry Pi-class CPU; the real limits are memory and power, and only a microcontroller rules it out. | I questioned it: "can't a small device run BEATs inference?" | Fixed. The hardware question became microcontroller vs. Linux board. |
| 10 | 22 Nov | Claude suggested evaluating with ROC AUC (following DCASE). | ROC AUC is misleading for such imbalanced data: many false alarms still show as a tiny false-positive rate. | I noticed it right away and suggested the precision-recall curve instead. | Fixed. PR-AUC is the main metric, with F-beta (beta 0.5) at the chosen threshold. |

## Template for new entries

`| # | date | what the AI said or did | what was wrong | how I caught it | status |`
