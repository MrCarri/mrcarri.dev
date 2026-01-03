---
title: "Building a Self-Hosted AI Application to Search Books"
date: 2026-01-03
categories: ["Software", "AI"]
tags: ["RAG", "LLM", "AI", "Manga", "Ollama"]
description: "How I built a local semantic search engine for manga using ChromaDB and Ollama."
draft: false
---

## The preamble

Half a year ago, I started working in a different company. I was a bit stagnant on the previous one for different reasons, and ended up changing sector. Looking back, it was an intelligent choice, and quickly adapted to the new environment. I started doing some AI agent development and integrations, and I actually enjoyed learning. And what really piqued my interest was how AI was able to retrieve customized results there weren't in the training: Retrieval-augmented generation.

In the meantime, I was enjoying my self-hosted manga reading instance when it struck me. I was running out of things to read, and my process of finding new things to read was a bit slow: First, filtering tags. Then, reading synopsis one by one, and then choosing what I actually sounds interesting and catches my attention.

My engineer nose tells this smells like something that cries automation... specifically, a semantic search engine powered by an LLM.

## The issue

Ok, we are onto something here. I tought about something:

  >How easily I can write a program that receives a user query, and finds the most similar books that match the query?
  
I did a bit of an investigation, and I made a list of things that we have to take into account.

1. A Crawler, or data Retrieval tool, to get information directly from the source, so we can build a dataset we can consult.
2. A Vector Database, to store and retrieve the data in a fast and organized way.
3. An Embedding model / query transformation to translate the user query into something that is actually intellegible to the database
4. A LLM / Generation model, to verbalize and reasons which results are actually related to the user query and generate a final output.

Then, I set a few restricitons:

1. Everything must be open source.
2. Everything must run locally on my RTX 2060 (6 GB VRAM).
3. Processing times must be reasonable short.


## The process
### Deciding the core tools

Ok, so it's crystal clear. Let's start with the easiest one: The Database. For this task, I chose ChromaDB becaise is an open source (Apache 2.0 license) non relational database that is designed specifically for storing embedding vectors, which actually are numeric representations of data in a multidimensional space. To make the definition less-wikipedia-like let's just say that anything can be converted into a N dimensional vector and then, this amount of numbers can be compared to other N vectors by similarity (a.k.a finding what is closest to it, using Vectorial Algebra.)

Another cool thing about ChromaDB is that it can be self-hosted, so I don't have to pay anything, And the cherry on top, turns out it has a great Python integration and documentation, so It's a clear match.

The embedding model that will work alongside ChromaDB is MiniLM-L6-v2. This is a really small model, that uses 384 dimensions and it's available on the python library called sentence-transformers. So, open source (Apache 2.0), and lightweight. It's task will be Vectorizing all data we need to embed into the database, and then the user query to compare.

Then, we need an LLM. For this, I chose to run this using open source models locally. They must fit on my GPU, so bigger models are out of question. I settled for around 4.5 GB or so in order to have a bigger margin when running the models. After a lot of tinkering around, I ended up with 2 models: llama3:8b and mistral:7b-instruct-q4_K_M, both within the security size, and trained to follow instructions.

### The workflow

How do all these pieces fit together? Let's look at it using real-world scenario. Imagine a user provides the following input:

  > I want a realistic medical mystery manga where a doctor or forensic professional solves crimes using biology and medicine.

We already have the database: The crawler has already requested data from mangadex and MiniLM has already converted all the data into 384 dimensional vectors and stored into the database using cosine similarity.

Now, we do the query. We must vectorize the user query. MiniLM converts the text into a vector, and we throw the query to the database. Chroma returns the top X results that are the most similar to the text input. In this case, I've set it to 15.

One consideration: This results might or might not be whatever the user wants. Reading from the query is very obvious. What the user wants is a thematic search. It wants books that feature medical mysteries, such as forensic science, or something related to biological mysteries. The embedding model doesn't know this, it just measuring similarity of words. How different are sentences.

This results are fed to the LLM. Then, it will make a recommendation of 5 books, and give the pertinent reasoning on why they have been chosen.

Real ouptut, using llama3:

```
llama3:8b Response: Based on the font context and user context, I recommend the following 5 manga that best match the query:

**RESULT 1:** Trace (Koga Kei)
This medical mystery manga features a rookie forensic researcher who teams up with a renowned expert to solve crimes using biology and medicine. The tags "Thriller", "Crime", "Medical", and "Police" make it an excellent match for the user's request.

**RESULT 2:** Sono Mogari wa Netsu wo Shiranai (Nicomichihiro)
This manga tells the story of a trainee doctor who partners with a detective to investigate various cases. The tags "Mystery" and "Medical" align well with the user's query, making it a suitable recommendation.

**RESULT 3:** Shin Onna Kensatsui (YADA Akira)
Asuka Kamijo, a female medical examiner, solves murders by examining and dissecting bodies, using her unique skills to uncover the truth. The tags "Action", "Mystery" match well with the user's request for a realistic medical mystery manga.

**RESULT 4:** Diagnosis From a Fallen Angel (Tsukasa Shiori)
This thriller follows nurse Reiko Mikazuki as she delivers punishment to criminals in a general hospital where cunning and heinous criminals are treated. The tags "Thriller", "Psychological", "Crime", and "Gore" align with the user's query, making it an excellent match.

**RESULT 5:** Dr. Prisoner (Ishikawa Komatsu)
This medical suspense action manga features a doctor who is closer to death than anyone else, solving murders using his unique skills. The tags "Action", "Psychological", "Drama", and "Medical" align well with the user's request.

These five manga best match the user's query for a realistic medical mystery manga that does not involve supernatural powers or magic.
```

### About results 

As you might have noticed, the query is a bit biased. I did a lot of tests with multiple queries, and also I had to improve the general quality of the inputs. I had to add to the embedding data the synopsis, and then also a few tags and the author. This is because synopses alone are not enough to provide a good result. Sometimes synopsis introduce a lot of noise and irrelevant text, and they actually don't match the tags at all. 

To demonstrate how the application works, I chose a very very niche result in order to illustrate how the LLM must filter the query results.

### Some considerations

You might be wondering: why use two different models? The main reason is Separation of Concerns.

The embedding model is a specialist: it is only responsible for vectorization. Both ChromaDB and the embedding model produce deterministic results (input A always produces B), but mathematical proximity doesn't always guarantee human relevance. This is where the LLM comes in.

To illustrate this, let’s look at a typical user query:

    "I want a realistic medical mystery manga where a doctor solves crimes using biology. It must be strictly realistic: NO ghosts, NO spirits, NO supernatural powers, and NO magic."

This is a "noisy" input. It’s full of negative constraints. Here lies the problem: Unless you are filtering metadata explicitly, an embedding model struggles to distinguish between "NO ghosts" and "YES ghosts". To the math, both sentences are semantically related to "ghosts".

As we have established in the preivous section, we are actually measuring is How similar is the user’s string to my vectorized data? Without the LLM acting as a "reasoning filter" at the end, the system might retrieve a supernatural manga simply because the word "ghost" appeared in the query.

Let's add some outputs that this query produces (before the LLM): 

  > [RESULT 1] Trace (Koga Kei). Tags: Thriller, Crime, Sexual Violence, Medical, Police, Mystery. Author: Koga Kei tags: Thriller, Crime, Sexual Violence, Medical, Police, Mystery year: 2016 synopsis: A crime suspense story set in a forensic laboratory where specialists work to uncover clues from seemingly insignificant bits of trace evidence. The story begins when Sawaguchi Nonna, a rookie forensic researcher, meets Mano Reiji, the man who solved her parents' murder case.


  > [RESULT 13] Mail (Yamazaki Housui). Tags: Thriller, Ghosts, Crime, Survival, Drama, Horror, Police, Supernatural, Mystery, Tragedy. Author: Yamazaki Housui tags: Thriller, Ghosts, Crime, Survival, Drama, Horror, Police, Supernatural, Mystery, Tragedy year: 1999 synopsis: Private detective Reiji Akiba has a theory about those awkward moments and weird coincidences we all encounter in life. They are actually encounters with the dead; their way of sending us a message. But you may not want to open such strange mail from beyond - not unless you can see the ghostly attachment, like Akiba can. And not unless you carry a gun that can kill what isn't alive, like Akiba's aptly named Kagutsuchi, "the tool between God and earth"… digging a divine grave to lay to rest the evil dead.

As you can see, the first result is the expected one. The second one, not so much. So, the task of the LLM is, ok, the user doesn't want this result, and some other results should rise to the top5. To get the best output, the query often requires some prompt tuning, moving away from natural conversational phrases towards a more keyword-centric or 'taggy' approach to minimize vector noise. For example:

```text
--- Manga recomendation expert initializing ---
Loading embedding model: all-MiniLM-L6-v2...
Searching for: 'mystery detective agency supernatural ghosts murder police investigation'

llama3:8b Response: Here are the 5 manga recommendations that best match the User context:

1. **Kyuubi - Choujou Jiken Taisakuka** by Nishijima Keita: This mystery manga features a man investigating paranormal phenomena, which aligns with the user's interest in supernatural mysteries.
2. **Ryoko's Case File** by Tanaka Yoshiki (田中芳樹): As an elite investigator, Ryoko Yakushiji solves bizarre and supernatural cases, making this manga a great match for the user's query.
3. **Mail** by Yamazaki Housui: With its focus on ghostly attachments and the protagonist's ability to see and interact with them, Mail is a great fit for the user's interest in supernatural mysteries.
4. **Genshou X: Choujou Genshou Sousaroku** by Onsen Nakaya: This manga explores paranormal events and crimes that defy logic, making it a suitable match for the user's query.
5. **Shikigami Twilight Days** by Iwashiro Toshiaki: As a one-shot manga about supernatural sleuthing, Shikigami Twilight Days aligns with the user's interest in mysteries involving ghosts and the paranormal.

Note that these recommendations are based solely on the provided font context and do not incorporate any external knowledge or assumptions. 
``` 

In this case, a I provided a tag input, and they match quite well. 

Let's add counterexamples that don't work so well to illustrate some future improvements. Here is a real unedited output from the system:
```text
--- Manga recomendation expert initializing ---
Loading embedding model: all-MiniLM-L6-v2...
Searching for: 'Author: Gosho Aoyama. Context: Mystery mangas investigation police thriller and crimes'

llama3:8b Response: Based on the font context, I recommend the following 5 manga that best match the User context:

1. **Detective Conan Special** by Gosho Aoyama: This manga is a spin-off series of short stories and cases from the Detective Conan universe. It features a high school detective who solves mysteries while investigating an organization that put him in this situation.
2. **Mori no Asagao Bangaihen - Shinjin Keimukan to Aru Shikeishuu no Monogatari** by Gouda Mamora: This manga is a side-story of the award-winning manga Mori no Asagao, which explores the issues of the death penalty through the friendship between a prison officer and an inmate on death row.
3. **Ice Blade: Diablo** by Takahashi Tsutomu: This manga features the case files of detectives at the Shinjuku Police Station, depicting the worst of human nature and the people that are the products of it.
4. **Akechi** by Iizuka Keita: This manga is a modern reimagining of the Edogawa Ranpo series, featuring a used bookstore owner who helps his childhood friend and detective solve crimes in Tokyo.
5. **Detective Story (Abe Yutaka)** by Abe Yutaka: This manga follows a young high school detective named Momoko Aoyama as she solves smaller, episodic cases since she took over her father's detective agency.

These recommendations are based on the tags and descriptions provided in the font context, which highlight themes of mystery, crime, police, and thrillers. 
```
```text
--- Manga recomendation expert initializing ---
Loading embedding model: all-MiniLM-L6-v2...
Searching for: 'Author: Naoki Urosawa. Context: Mystery mangas investigation police thriller and crimes'

llama3:8b Response: Based on the analysis of manga descriptions and tags provided in the Font context, I recommend the following 5 manga that best match the User context:

**1. Sneeze!: Naoki Urasawa Story Collection (Urasawa Naoki)**
This collection of short stories by Naoki Urasawa matches the user's query due to its thriller and mystery elements, which are common themes in Urasawa's works.

**2. Mujirushi - The Sign of Dreams (Urasawa Naoki)**
This miniseries also matches the user's query because it features a mix of thriller, comedy, crime, drama, and mystery elements, all hallmarks of Naoki Urasawa's style.

**3. Goodbye Mr. Bunny (Urasawa Naoki)**
Another work by Naoki Urasawa, this one-shot manga fits the user's query due to its action-packed storyline with elements of crime, drama, and police themes.

**4. DAMiYAN! (Urasawa Naoki)**
This one-shot manga also matches the user's query because it features a mix of thriller, comedy, crime, mafia, magic, gore, and fantasy elements, all common in Urasawa's works.

**5. Meitantei Shiro Series (Serizawa Naoki)**
This series fits the user's query due to its action-packed storyline with elements of comedy, police, and mystery themes, which are all reminiscent of Naoki Urasawa's style. 
```

In this case, I've given names of famous authors. As you can see, the results for Gosho Aoyama are far worse from the ones of Urasawa Naoki, but both are quite bad. If you are familiar with these authors, you will realize that most of the important works are quite not present. The LLM can't do miracles, it has a limited context window and the data is what it is. A few future improvements can be improving the input descriptions or adding a **hybrid search** to combine the database results with traditional tag-filtering before the vector database retrieval and the final LLM response.

## Wrap up

So, to close this article, we have a working architecture. A crawler, a vector database and LLM to filter. On paper, the system should work, taking user intent into account in addition to word-matching. In practice, the system is not perfect. As you might have noticed, the quality of the search results is closely related to the quality of the prompt. Actually, using a human-phrase input prompt leads to worse results. For example, if you start with "I want a manga that" if a vector contains this start, it will rank higher in the search even if it doesn't have any relation with the rest of the query. But overall, I think it's a fun experiment. 

You will have the whole source code on my  [Github](https://github.com/MrCarri), along with some instructions. In a future article, I want to work  a bit on embedding and euclidian vs cosine searches, and some filtering. Stay tuned!
