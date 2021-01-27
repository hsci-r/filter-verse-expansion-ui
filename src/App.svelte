<svelte:options immutable />
<script lang="typescript">
  import PapaParse from "papaparse"
  import { onMount } from "svelte"
  import produce from "immer"
  import type { Draft } from 'immer'

  const api = new URL("https://filter-octavo.rahtiapp.fi/filter/search")

  const asyncParseCsv = (url: string) =>
  new Promise<PapaParse.ParseResult<{ text: string }>>((resolve, reject) =>
    PapaParse.parse<{ text: string }>(url, {
      download: true,
      header: true,
      complete: results => resolve(results),
      error: err => reject(err)
    })
  )

  type IVerse = Readonly<{
    verse: string
    selected: boolean
    normalizedVerse: string | undefined
    verseIdSet: ReadonlySet<string>
    verseIds: ReadonlyArray<string>
    queryIndex: number
    verseIndex: number
  }>

  type IQueryWord = Readonly<{
    word: string
    selected: boolean
    expand: number
    queryIndex: number
    queryWordIndex: number
  }>

  type IQuery = Readonly<{
    query: string
    verses: ReadonlyArray<IVerse>
    queryWords: ReadonlyArray<IQueryWord>
    queryIndex: number
  }>

  type IQueryState = Readonly<{
    seenQueryWords: ReadonlySet<string>
    verseDict: ReadonlyMap<string,IVerse>
    queries: ReadonlyArray<IQuery>
  }>

  const mutate = <T,>(cur: T, transform: (state: Draft<T>) => void) => produce(cur, draft => { transform(draft) })

  let minScore = 0
  let additionalQuery = " AND +verse_type:V"
  let currentQuery = ""
  $: currentQuery = "(" +
    queryState.queries
      .flatMap(q => q.queryWords)
      .filter(qw => qw.selected)
      .map(qw => qw.word + "~" + qw.expand)
      .join(" OR ") +
    ") " +
    additionalQuery

  const clusterCache = new Map<string,PapaParse.ParseResult<{ text: string }>>()

  let queryState: IQueryState = {
    seenQueryWords: new Set<string>(),
    verseDict: new Map<string,IVerse>(),
    queries: []
  }

  let selectedVerses: ReadonlyArray<IVerse>

  $: selectedVerses = queryState.queries.flatMap(q => q.verses).filter(v => v.selected)

  let dqueries: ReadonlyArray<IQuery>

  $: dqueries = queryState.queries

  let tasks = 0

  let initialQuery = ""

  let error: string

  const fetchAndParseClusters = async(url: string, queryData: Draft<IQuery>, nverseDict: Map<string,IVerse>, nverseIdsDict: Map<string,string>) => {
    let csv
    if (clusterCache.has(url)) 
      csv = clusterCache.get(url)
    else {
      csv = await asyncParseCsv(url)
      if (csv.errors.length>0 && csv.errors[0].type=="Delimiter") throw Error("Resource does not look like a CSV")
      clusterCache.set(url,csv)
    } 
    for (const row of csv.data)
      if (row["text"]) {
        const verse = row["text"]
        const verse_id = row["nro"] + "_" + row["pos"]
        if (!queryState.verseDict.has(verse)) { 
          if (!nverseDict.has(verse)) {
            const nverse: Draft<IVerse> = {
              verse,
              selected: true,
              normalizedVerse: undefined,
              verseIdSet: new Set([verse_id]),
              verseIds: [verse_id],
              queryIndex: -1,
              verseIndex: queryData.verses.length
            }
            queryData.verses.push(nverse)
            nverseDict.set(verse,nverse)
          } else if (!nverseDict.get(verse).verseIdSet.has(verse_id)) {
            nverseDict.get(verse)!.verseIdSet.add(verse_id)
            nverseDict.get(verse)!.verseIds.push(verse_id)
          }
        } else if (!queryState.verseDict.get(verse)!.verseIdSet.has(verse_id))
          nverseIdsDict.set(verse_id,verse)
      }    
  }

  const hashChanged = async () => {
    const params = new URLSearchParams(window.location.hash.slice(1))
    if (params.has("s")) {
      initialQuery = params.get("s")
      tasks++
      queryState = mutate(queryState, qs => {
        qs.verseDict.clear()
        qs.seenQueryWords.clear()
        qs.queries = []
      })
      const queryData: Draft<IQuery> = {
        query: "Initial",
        verses: [],
        queryWords: [],
        queryIndex: 0
      }
      const nverseDict = new Map<string,Draft<IVerse>>()
      const nverseIdsDict = new Map<string,string>()
      if (initialQuery.startsWith("http")) {
        try {
          await fetchAndParseClusters(initialQuery,queryData,nverseDict,nverseIdsDict)
          error = undefined
        } catch (err) {
          error = "Couldn't parse CSV from URL "+initialQuery+". Parser returned: "+err
          throw err
        } finally {
          tasks--
        }
      } else {
        error = undefined
        const verse: Draft<IVerse> = {
              verse: initialQuery,
              selected: true,
              normalizedVerse: undefined,
              verseIds: [],
              verseIdSet: new Set(),
              queryIndex: 0,
              verseIndex: 0
            }
        queryData.verses = [verse]
        nverseDict.set(initialQuery,verse)
        tasks--
      }
      let nminScore = Number.MAX_SAFE_INTEGER
      for (const verse of queryData.verses) {
        const cminScore = verse.verse.split(" ").length
        if (cminScore < nminScore) nminScore = cminScore
      }
      minScore = nminScore
      queryState = mutate(queryState, qs => {
        for (const verse of queryData.verses) verse.queryIndex = 0
        qs.queries = [queryData]
        for (const verse of nverseDict.values())
          qs.verseDict.set(verse.verse,verse)
        for (const [verseId,verse] of nverseIdsDict.entries()) {
          const mverse = qs.verseDict.get(verse)
          mverse.verseIdSet.add(verseId)
          mverse.verseIds.push(verseId)
        }
      })
      extractQueryWords(queryData)
    }
  }
  onMount(hashChanged)

  const verseToQueryWords = (verse: string) =>
  verse
    .split(" ")
    .map(word => word.toLowerCase().replace(/\P{L}+/gu, ""))
    .filter(word => word.length > 0)

  const extractQueryWords = (iquery: IQuery) => queryState = mutate(queryState,qs => {
    const query = qs.queries[iquery.queryIndex]
    for (const queryWord of query.queryWords) qs.seenQueryWords.delete(queryWord.word)
    query.queryWords = []
    for (const verse of query.verses)
      if (verse.selected)
        for (const word of verse.normalizedVerse
          ? verse.normalizedVerse.split(" ")
          : verseToQueryWords(verse.verse)) {
          if (!qs.seenQueryWords.has(word)) {
            const queryWord: IQueryWord = {
              word: word,
              selected: word.length > 2,
              expand: word.length < 3 ? 0 : word.length < 6 ? 1 : 2,
              queryIndex: iquery.queryIndex,
              queryWordIndex: query.queryWords.length
            }
            qs.seenQueryWords.add(word)
            query.queryWords.push(queryWord)
          }
        }
  })

  const removeQuery = (query:IQuery) => queryState = mutate(queryState,qs => {
      qs.queries.splice(query.queryIndex, 1)
      for (const queryWord of query.queryWords)
        qs.seenQueryWords.delete(queryWord.word)
      for (const verse of query.verses) qs.verseDict.delete(verse.verse)
      for (const mquery of qs.queries.slice(query.queryIndex)) {
        mquery.queryIndex--
        for (const verse of mquery.verses)
          verse.queryIndex--
        for (const queryWord of mquery.queryWords)
          queryWord.queryIndex--
      }
    })

  const doQuery = async () => {
    tasks++
    const mcurrentQuery = currentQuery
    const p = new URLSearchParams({
      minScore: "" + minScore,
      field: "content",
      level: "verse",
      query: currentQuery,
      limit: "-1"
    })
    p.append("field", "verse_id")
    p.append("field", "normalized_verse")
    api.search = p.toString()
    const r = await fetch(api.toString())
    const d: {
      result: {
        docs: {
          content: string
          normalized_verse: string
          verse_id: string
        }[]
      }
    } = await r.json()
    queryState = mutate(queryState,qs => {
      const query: Draft<IQuery> = {
        query: mcurrentQuery,
        verses: [],
        queryWords: [],
        queryIndex: qs.queries.length
      }
      for (const result of d.result.docs) {
        const verse = result.content
        if (!qs.verseDict.has(verse)) {
          const verseData: Draft<IVerse> = {
            verse: verse,
            selected: true,
            normalizedVerse: result.normalized_verse,
            verseIdSet: new Set([result.verse_id]),
            verseIds: [result.verse_id],
            queryIndex: qs.queries.length,
            verseIndex: query.verses.length
          }
          query.verses.push(verseData)
          qs.verseDict.set(verse, verseData)
        } else if (!qs.verseDict.get(verse)!.verseIdSet.has(result.verse_id)) {
          qs.verseDict.get(verse)!.verseIdSet.add(result.verse_id)
          qs.verseDict.get(verse)!.verseIds.push(result.verse_id)
        }
      }
      qs.queries.push(query)
    })
    tasks--
  }

  const toggleVerses  = (iquery: IQuery) => queryState = mutate(queryState,qs => {
    const query = qs.queries[iquery.queryIndex]
    for (const verse of query.verses) verse.selected=!verse.selected
  })

  const toggleVerse = (verse: IVerse) => queryState = mutate(queryState,qs => qs.queries[verse.queryIndex].verses[verse.verseIndex].selected = !verse.selected)
  const toggleQueryWord = (queryWord: IQueryWord) => queryState = mutate(queryState, qs => qs.queries[queryWord.queryIndex].queryWords[queryWord.queryWordIndex].selected = !queryWord.selected)
  const setExpand = (queryWord: IQueryWord,expand: number) => queryState = mutate(queryState, qs => qs.queries[queryWord.queryIndex].queryWords[queryWord.queryWordIndex].expand = expand)

  const fetchClusters = async() => {
    tasks++
    const queryData: Draft<IQuery> = {
      query: "Clusters for ",
      verses: [],
      queryWords: [],
      queryIndex: -1
    }
    const nverseDict = new Map<string,Draft<IVerse>>()
    const nverseIdsDict = new Map<string,string>()
    const promises: Promise<void>[] = []
    for (const query of queryState.queries) 
      for (const verse of query.verses)
        if (verse.selected) {
          queryData.query += '"' + verse.verse + '", '
          for (const verseId of verse.verseIds)
            promises.push(fetchAndParseClusters("https://ldf.fi/corsproxy/runoregi.rahtiapp.fi/verse?format=csv&nro=" +
          verseId.split("_").join("&pos="),queryData,nverseDict,nverseIdsDict))
        }
    try {
      await Promise.all(promises)
    } catch (err) {
      error = "Couldn't parse CSV for at least one verse. Parser returned: "+err
      throw err
    } finally {
      tasks--
    }
    queryData.query = queryData.query.slice(0, -2)
    queryState = mutate(queryState, qs => {
      queryData.queryIndex = qs.queries.length
      for (const verse of queryData.verses) verse.queryIndex = qs.queries.length
      qs.queries.push(queryData)
      for (const verse of nverseDict.values())
        qs.verseDict.set(verse.verse,verse)
      for (const [verseId,verse] of nverseIdsDict.entries()) {
        const mverse = qs.verseDict.get(verse)
        mverse.verseIdSet.add(verseId)
        mverse.verseIds.push(verseId)
      }
    })
  }
</script>

<svelte:window on:hashchange={hashChanged} />

<template lang="pug">
  .px-4.mx-auto(class="lg:container")
    h1.text-2xl.my-3.font-bold FILTER verse expansion UI
    label.block.max-w-2xl Initial Query
      input.rounded.w-full.mt-1(type="text",value="{initialQuery}" on:keyup!="{e => { if (e.code == 'Enter') window.location.hash='s='+encodeURIComponent(e.currentTarget.value) }}" size="40")
    h2.text-xl.mt-3.font-bold Results
    .flex.flex-col.mb-2
      +each("dqueries as query"): .border-2.rounded-md.px-2.pb-2.pt-1.my-1
        .flex.justify-between.mt-2.mb-1.pb-1.border-b
          .flex(style="min-width:0")
            h3.text-lg.truncate {query.query} 
            h3.text-lg.ml-1 ({query.verses.length}) 
          button.w-6.h-6.flex-shrink-0(on:click!="{() => removeQuery(query)}")
            svg.text-gray-500(viewBox="0 0 20 20" fill="currentColor")
              path(fill-rule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z" clip-rule="evenodd")
        .flex.justify-start.divide-x.pt-1
          .pr-4
            button.block.btn(on:click!="{() => toggleVerses(query)}") Toggle all
            .flex.flex-col: +each("query.verses as verse"): .flex.flex-wrap
              label.block.select-none.font-mono
                input.mr-1(type="checkbox" checked="{verse.selected}" on:change!="{() => toggleVerse(verse)}")
                | {verse.verse}
              +each("verse.verseIds as verseId,i"): .block.bg-indigo-500.text-white.text-center.rounded-full.text-sm.px-2.py-1(class="m-0.5"): a(href!="http://runoregi.rahtiapp.fi/poem?nro={verseId.split('_').join('&hl=')+'#'+verseId.split('_')[1]}" target="_blank") {verseId.split('_').join("/")}
            button.block.btn(disabled!="{tasks>0}" on:click!="{e => extractQueryWords(query)}") Extract query words
          .pl-4.flex.flex-col.min-w-max: +each("query.queryWords as queryWord"): label.block.select-none
              input(type="checkbox" checked="{queryWord.selected}" on:change!="{() => toggleQueryWord(queryWord)}")
              | {queryWord.word}~
              input.inline.p-0.border-0(type="number" min="0" max="2" value="{queryWord.expand}" on:change!="{e => setExpand(queryWord,e.currentTarget.valueAsNumber)}")
    +if('error'): .text-xl.text-red-500 Error: {error}
    +if('tasks > 0'): .flex.justify-center: svg.animate-spin.h-10.w-10(fill="none" viewBox="0 0 24 24")
      circle.opacity-25(cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4")
      path.opacity-75(fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z")
    h3.block.text-lg.mt-2.font-bold Expand verses through Runoregi clusters
    button.block.btn(on:click!="{fetchClusters}" disabled!="{tasks > 0}") Expand {selectedVerses.length} verses
    h3.block.text-lg.mt-2.font-bold Search for verses through Octavo
    .border-2.rounded-md.px-2.pb-2.pt-1.flex.flex-col.mb-2
      .flex.justify-start.items-end
        label.block.mr-1.mb-1 Additional query restrictions
          input.rounded.w-full.mt-1(type="text" bind:value="{additionalQuery}")
        .flex.flex-col
          label.block.mb-1 Minimum words required
            input.block.rounded.mt-1(type="number" min="0" max="99" bind:value="{minScore}")
      textarea.block.max-2-2xl(bind:value="{currentQuery}")
      div
        button.mt-1.btn(on:click="{doQuery}" disabled!="{tasks > 0}") Search      
    h3.block.text-lg.mt-2.font-bold Results ({selectedVerses.length} currently selected verses)
    table.mb-2
      tr.border-b
        th Verse
        th Verse instances
      +each("selectedVerses as verse"): tr.border
        td.whitespace-nowrap.border.align-top.p-1.font-mono {verse.verse}
        td.border.align-top.p-1.font-mono: +each("verse.verseIds as verseId,i") 
          a(href!="http://runoregi.rahtiapp.fi/poem?nro={verseId.split('_').join('&hl=')+'#'+verseId.split('_')[1]}" target="_blank") 
                | {verseId.split('_').join("/")}
                +if("i < verse.verseIds.length - 1") {@html ', '}
</template>

<style lang="sugarss">
  .btn
	  @apply px-2 py-1 bg-transparent text-blue-600 font-semibold hover:bg-blue-500 hover:text-white border border-blue-500 hover:border-transparent rounded disabled:opacity-50 disabled:bg-transparent disabled:text-blue-600 disabled:border-blue-600
</style>
