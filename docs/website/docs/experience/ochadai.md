# お茶の水女子大学

!!! note ""
    Visit the [official website](https://www.ocha.ac.jp/en/).

Ochanomizu University is a women-only university in Tokyo, Japan.

In 2019, I joined a computer science laboratory for three months as an intern during my studies at the [IUT de Bordeaux](iut.md) (men are allowed among researchers). This laboratory was working on natural language understanding, using grammar and vocabulary cues. Their flagship project, [`ccg2lambda`](https://github.com/mynlp/ccg2lambda), allows converting English or Japanese sentences into mathematical expressions, which can be fed into a solver to prove or disprove.

Along two other students from France, our task was to adapt these mathematical expressions into SPARQL, [Wikidata's query language](https://www.wikidata.org/wiki/Wikidata:SPARQL_tutorial). In its final version, the project allows transcribing logical combinations ("and", "or"…) and some logical constraints ("in" followed by a date range) into queries as well as answering direct questions (e.g. "Which country won the Football World Cup in 1998"). However, the system is weak to homographs and has to ask the user to disambiguate. The repository is [available here](https://github.com/clovis-ai/ccg2lambda-qa-assistant).
