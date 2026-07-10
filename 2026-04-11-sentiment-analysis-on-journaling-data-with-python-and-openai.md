---
title: Sentiment Analysis on Journaling Data with Python and OpenAI
slug: sentiment-analysis-on-journaling-data-with-python-and-openai
date_published: 2026-04-11T07:55:00.000Z
date_updated: 2026-04-17T11:55:14.000Z
excerpt: This experiment uses journaling data to perform sentiment and emotion analysis via the OpenAI Python API. Results show a bias toward negative emotions, indicating entries are more often written during high-intensity states. Outputs are non-deterministic and should be interpreted cautiously.
---

### Background

Last month I was trying to get to know a girl. She mentioned that she does daily journaling, so I decided to start journaling as well so we would have something in common to talk about. It’s been around 15 days since I started, and I want to write about this experience as fairly as possible from a non-medical perspective.

---

### Introduction

Both of my friends who consulted a psychologist were encouraged to start journaling or note-taking. I think this aligns with some research showing that journaling has been associated with improvements in mental and physical health by reducing anxiety, lowering stress, and promoting adaptive coping mechanisms [1]. Journaling is also considered a common non-pharmacological tool in managing mental health issues [2]. Based on these points, it seems reasonable that psychologists often suggest journaling.

However, at this point I realized something important: I cannot directly assume that what the psychologist suggested was strictly “journaling” or “writing therapy”, because these terms have different meanings and purposes.

Writing therapy refers to writing as a method to express and reflect on oneself, often used as a tool for healing and personal growth [3]. It usually involves structured reflection such as “why it matters” and “what it means”. On the other hand, journaling is more general and does not require specific instructions or methodologies [3]; in practice, it often focuses on recording what happened. There is also expressive writing, which focuses on putting feelings and thoughts into written words to cope with stressful or traumatic situations [3].

---

### Tools Used

Since my background is in DevOps engineering, I used tools I was already familiar with. I wrote my journal entries in [Docmost](https://docmost.com/), which I had previously used for work documentation. I separated entries by page, with one page per day/date.

At the beginning, I didn’t take this project too seriously, so I also included my daily to-do lists in the same page. This likely introduced unnecessary noise into the data when performing analysis later.

For analysis, I used Python along with the OpenAI API to process and extract sentiment from the journal entries.

---

### Analyzing the Emotions

As mentioned earlier, I used Python to call the OpenAI API and analyze the journal data stored in JSON format. The goal was to extract structured information such as sentiment, emotions, and a short reason for each entry.

Because the data was stored in Docmost, I first exported it by dumping the PostgreSQL database into JSON format so it could be processed by the script.

### Exporting the Docmost Data

    SELECT text_content, title
    FROM public.pages
    WHERE parent_page_id IN (
        '019d530a-8086-7daa-bf70-5944cc4c6f6d', -- April
        '019d2141-0c8b-79d3-8c11-e750ef994ecc'  -- March
    )
    ORDER BY title ASC;
    

### Sentiment Analysis Script

    import json
    import os
    import re
    from openai import OpenAI
    from tqdm import tqdm
    
    client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
    
    INPUT_FILE = "journal.json"
    OUTPUT_FILE = "journal_sentiment.json"
    
    
    def extract_json(text):
        try:
            return json.loads(text)
        except:
            match = re.search(r'\{.*\}', text, re.DOTALL)
            if match:
                return json.loads(match.group())
            else:
                raise ValueError("No JSON found in response")
    
    
    def analyze_sentiment(text):
        text = text[:2000]
    
        prompt = f"""
    Analyze this journal entry.
    
    Return ONLY JSON:
    {{
      "sentiment": "positive|neutral|negative",
      "score": number (-1 to 1),
      "emotions": ["happy","sad","lonely","stressed","angry","anxious","excited","burnout","neutral"],
      "reason": "max 12 words"
    }}
    
    Text:
    \"\"\"{text}\"\"\"
    """
    
        try:
            response = client.chat.completions.create(
                model="gpt-4.1-mini",
                messages=[
                    {"role": "system", "content": "Return strictly JSON only."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0
            )
    
            content = response.choices[0].message.content.strip()
            return extract_json(content)
    
        except Exception as e:
            return {
                "sentiment": "error",
                "score": 0,
                "emotions": [],
                "reason": str(e)
            }
    
    
    def main():
        with open(INPUT_FILE, "r") as f:
            data = json.load(f)
    
        entries = list(data.values())[0]
        results = []
    
        for entry in tqdm(entries):
            text = entry.get("text_content", "")
            title = entry.get("title", "")
    
            sentiment = analyze_sentiment(text)
    
            results.append({
                "date": title,
                "text": text,
                "sentiment": sentiment.get("sentiment"),
                "score": sentiment.get("score"),
                "emotions": sentiment.get("emotions"),
                "reason": sentiment.get("reason")
            })
    
        with open(OUTPUT_FILE, "w") as f:
            json.dump(results, f, indent=2)
    
        print(f"Done. Saved to {OUTPUT_FILE}")
    
    
    if __name__ == "__main__":
        main()

### Example Output

    {
      "date": "March 31, 2026",
      "text": "...",
      "sentiment": "neutral",
      "score": 0,
      "emotions": ["sad", "stressed"],
      "reason": "Missed chatting, but completed tasks and paper submission"
    }
    

### Aggregating Emotion Data

To make the results easier to interpret, I aggregated all emotions and counted their occurrences:

    import json
    from collections import Counter
    
    with open("journal_sentiment.json", "r") as f:
        data = json.load(f)
    
    counter = Counter()
    
    for entry in data:
        emotions = entry.get("emotions", [])
        counter.update(emotions)
    
    for emotion, count in counter.most_common():
        print(f"{emotion}: {count}")

### Example Output

    stressed: 14
    anxious: 12
    sad: 4
    neutral: 4
    burnout: 4
    lonely: 2
    angry: 2
    excited: 2
    focused: 1
    hopeful: 1
    frustrated: 1
    happy: 1
    

---

### Results and Discussion

Based on around 15 days of journaling data, the top three emotions were negative (stressed, anxious, sad). Since the data source is my own writing, I can explain this: this suggests my journaling behavior is reactive rather than reflective, as I tend to write when processing negative emotions instead of experiencing positive ones. This behavior aligns more with expressive writing than general journaling.

Another possibility is that I prefer to share positive experiences directly with friends instead of writing them down. There is also research suggesting that listening to happy stories can improve people’s mood [4]. For a more balanced analysis, it would be better to also include positive experiences in the journal.

---

### Conclusion

Besides the initial motivation of getting closer to someone, daily journaling helped me express my emotions more clearly. It also served as a place to track my daily tasks.

For future improvements, if the goal is self-understanding, journaling needs to capture both positive and negative experiences. It should intentionally include at least one positive and one negative reflection per entry.

From a technical perspective, using LLMs for emotion analysis has limitations. The outputs are not fully consistent due to the generative nature of the models. Even with the same input and model, results may vary. Future work could explore more deterministic approaches using pretrained NLP models such as IndoBERT.

Additionally, my writing style is inconsistent, including typos and slang, which may affect analysis accuracy. Standardizing writing style could improve results.

---

### References

[1] G. Raj, J. Jayson, P. Deshpande, and N. Sood, “Theoretical and practical aspects of modern psychology,” ch. 3, 2025.

[2] M. Sohal, P. Singh, B. S. Dhillon, and H. S. Gill, “Efficacy of journaling in the management of mental illness: A systematic review and meta-analysis,” Fam. Med. Community Health, vol. 10, no. 1, p. e001154, Mar. 2022, doi: [10.1136/fmch-2021-001154](https://doi.org/10.1136/fmch-2021-001154).

[3] C. Ruini and C. C. Mortara, “Writing technique across psychotherapies—From traditional expressive writing to new positive psychology interventions: A narrative review,” J. Contemp. Psychother., vol. 52, no. 1, pp. 23–34, Mar. 2022, doi: [10.1007/s10879-021-09520-9](https://doi.org/10.1007/s10879-021-09520-9).

[4] K. I. Batcho, “When nostalgia tilts to sad: Anticipatory and personal nostalgia,” Front. Psychol., vol. 11, p. 1186, May 2020, doi: [10.3389/fpsyg.2020.01186](https://doi.org/10.3389/fpsyg.2020.01186).
