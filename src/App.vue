<script setup lang="ts">
/*****************************************
 **************** IMPORTS ****************
 ****************************************/

import { computed, ref, watchEffect } from "vue"

/********************************************
 **************** INTERFACES ****************
 *******************************************/

/**
 * Candidate data needed for process
 */
interface candidate {
    id: number
    passed_karat: boolean
    scores: number[]
    scores_raw: number[]
    score: number
    candidate: string
    flag: boolean
    modules: string[]
    client_rec: string
}

/**
 * Partial list of field from Metabase question https://metabase-pii.karat.io/question/335-ranked-candidates-with-outcomes.
 */
interface raw_candidate {
    candidate: string
    score: number
    original_score: null|number
    knowledge_question: string
    kq_scaled_scores: string
    primary_question: string
    client_rec: string
    candidacy_state:
        'Offer accepted'|
        'Offer outstanding'|
        'Declined at on-site interview'|
        'Invited on-site interview'|
        'Declined at Karat interview'|
        'Inactive'|
        'Needs decision'|
        'Declined at phone screen'|
        'Invited to phone screen'
}

interface domain_list {
    content: string
    count: number
    map: number
}
interface candidate_data {
    passed_karat: boolean
    scores: number[]
    scores_raw: number[]
    score: number
    candidate: string
    flag: boolean
    modules: string[]
    client_rec: string
}
interface results {
    weights: number[]
    dnp: number
    itnr: number
    candidates: result_candidate[]
    auto_dnp: number[]
}
interface result_candidate {
    id: number
    name: string
    system_rec: 'DNP' | 'RFR' | 'ITNR'
    client_rec: string
    score: number
    scores: number[]
    pass_expectation: number
    passed_karat: boolean
}

/****************************************************
 **************** COMPUTED VARIABLES ****************
 ***************************************************/

const sorted_candidate_list = computed(() => candidate_list.value.slice().sort((a, b) => a.flag == b.flag ? (a.candidate < b.candidate ? -1 : 1) : a.flag ? -1 : 1))
const max_modules_used = computed(() => candidate_list.value.reduce((pv, cv) => Math.max(pv, cv.modules.length), 0))
const next_button_disabled = computed(() => {
    if(app_error.value) return true
    switch(state.value){
        case 'upload': return raw_candidate_data.value.length === 0
        case 'edit_domains': return domains.value.some(item => item == '')
    }
    return false
})
const alignment_confidence_class = computed(() => {
    const confidence = 1 - anomolus_candidates.value.length / results.value.candidates.length
    if(confidence < .8) return 'color-red'
    if(confidence < .9) return 'color-orange'
    return'color-green'
})
const anomolus_candidates = computed(() => results.value.candidates
    .filter(item => item.pass_expectation >= 75 && !item.passed_karat|| item.pass_expectation <= 25 && item.passed_karat)
    .sort((a, b) =>  Math.abs(2 * a.pass_expectation - 100) > Math.abs(2 * b.pass_expectation - 100) ? -1 : 1
    )
)

/****************************************************
 **************** REACTIVE VARIABLES ****************
 ***************************************************/

const header = ref<HTMLElement>(null)
const header_height = ref<string>(`0px`)

/**
 * Reactive variable that stores the state of the process
 */
const state = ref<'upload'|'clean_candidate_list'|'edit_domains'|'view_results'|'view_alignment'>('upload')

/**
 * Reactive variable used to store json data uploaded from Metabase question https://metabase-pii.karat.io/question/335-ranked-candidates-with-outcomes.
 */
const raw_candidate_data = ref('')

/**
 * Reactive variable used to store a list of candidates extracted from the raw_candidate_data
 */
const candidate_data = ref<candidate_data[]>([])

/**
 * Reactive variable used to store the domains that the bars will be based on
 */
const domains = ref<string[]>([])
const role = ref('')
const app_error = ref('')
const results = ref<results>({
    dnp: 0,
    itnr: 0,
    weights: [],
    candidates:[],
    auto_dnp: []
})

/**
 * Reactive variable used to store the content used by candidates
 */
const domain_list = ref<domain_list[]>([])

/**
 * Reactive variable used to store the candidates that bars will be based on
 */
const process_list = ref<candidate[]>([])

const ignore = ref(100)

const candidate_list = ref<candidate[]>([])

const processing = ref<boolean>(false)

/******************************************
 **************** WATCHERS ****************
 *****************************************/

// watches: header
// note   : This initializes the resizeObserver when the header ref changes
const resizeObserver = new ResizeObserver(entries => {
    for (let entry of entries) {
        header_height.value = entry.contentRect.height + 'px'
    }
})
watchEffect(() => {
    if(header.value === null) return
    resizeObserver.observe(header.value)
})

// watches: raw_candidate_date
// updates: candidate_list
// updates: app_error
watchEffect(() => {
    try {
        let anonymous_candidate_count = 0
        let id = 0
        if(raw_candidate_data.value == ``){
            app_error.value = ``
            candidate_list.value = []
            return
        }

        const raw_candidate_list:raw_candidate[] = JSON.parse(raw_candidate_data.value)
        candidate_list.value = raw_candidate_list

        // filter out candidates that had a redo - since we don't know how the redo influenced their results
        .filter(raw_candidate => raw_candidate.original_score === null)

        // filter out candidates where the client hasn't made a decision to pass or decline
        .filter(raw_candidate => ['Inactive','Needs decision'].includes(raw_candidate.candidacy_state) == false)

        // Map raw data into the desired form
        .map(raw_candidate => {
            // Store a list of the knowledge questions used
            const modules = raw_candidate.knowledge_question ? raw_candidate.knowledge_question.split(',') : []

            // Store a seperate list of knowledge question scores
            const scores = raw_candidate.kq_scaled_scores.length ? raw_candidate.kq_scaled_scores.split(',').map(score => parseInt(score)) : []

            // Determine if coding was used by checking for language. Sometimes language has a value but shouldn't when this happens check if coding score
            // is available and if it isn't also check if the score matched the first knowledge score. If it does then the coding likely didn't occure.
            if(modules.includes(raw_candidate.primary_question) == false){
                scores.unshift(raw_candidate.score)
                modules.unshift('Coding')
            }
            
            const candidate:candidate = {
                id: ++id,
                passed_karat: raw_candidate.candidacy_state != 'Declined at Karat interview',
                scores,
                scores_raw: scores,
                score: 0,
                candidate: raw_candidate.candidate == '<User Requested Removal>' ? `Anonomus Candidate ${++anonymous_candidate_count}` : raw_candidate.candidate,
                flag: false,
                modules,
                client_rec: raw_candidate.client_rec
            }

            app_error.value = ``

            return candidate
        })
    }
    catch(e){
        app_error.value = `Your JSON data is invalid.`
        candidate_list.value = []
    }
})

// watches: candidate_list
// watches: domain_list
// updates: domain_list
watchEffect(() => {
    const content_list = [...new Set(candidate_list.value.map(candidate => candidate.modules).flat())]
    const list:domain_list[] = []
    content_list.forEach(content => {
        list.push({
            content,
            count: candidate_list.value.filter(candidate => candidate.modules.includes(content)).length,
            map: domain_list.value.find(domain => domain.content == content)?.map ?? 0
        })
    })
    domain_list.value = list
})

// watches: candidate_list
// watches: state
// updates: candidate_list -> flag
// updates: app_error
watchEffect(() => {
    // Only runs in when the candidate list needs to be cleaned
    if(state.value != 'clean_candidate_list') return

    // flag stores a list of candidates based on how many modules they have scored
    let flag:candidate_data[][] = []
    candidate_list.value.forEach(item => {
        flag[item.scores.length] ??= []
        flag[item.scores.length].push(item)
    })

    // Filter out empty items in flag list
    flag = flag.filter(item => item.length !== undefined)

    // Cycle through candidate list to clear the error flag
    candidate_list.value.forEach(candidate => candidate.flag = false)

    // Get a list of candidates that have an undefined score
    const no_score_list = candidate_list.value.filter(candidate => candidate.scores.some(score => score === null))

    // Set error flag for each candidate in the no score list
    no_score_list.forEach(item => item.flag = true)

    let error = ''

    // If there's more than on score length in the flag an error occured
    if(flag.length > 1){
        // Get the score length that is the one that is most likely valid while removing that list from the flag list
        const most_likely = flag.pop()[0].scores.length

        // Flaten the flag list since all remainning are likely to be errors
        flag.flat().forEach(item => item.flag = true)
        error += 'Candidates all need to have the same number of modules scored'
    }

    // If any scores are undefined an error occured
    if(no_score_list.length){
        if(error) error += ' - '
        error += 'Content map has created null scores'
    }
    app_error.value = error
})

// watches: candidate_list
// updates: candidate_list -> scores
watchEffect(() => {
    candidate_list.value.forEach(candidate => {
        let i = 0

        // Scores are filled with null to allow scores to be checked with .some
        const scores:number[] = Array(candidate.modules.length).fill(null)
        candidate.modules.forEach(module => {
            const n = domain_list.value.filter(domain => domain.content == module)[0].map
            scores[n] = candidate.scores_raw[i++]
        })

        // Remove unused scores on the end of the scores array
        i = scores.length
        while(scores[--i] === null) scores.pop()

        // set scores
        candidate.scores = scores
    })
})

// watches: candidate_list
// updates: domains
watchEffect(() => {
    if(state.value != 'clean_candidate_list') return

    // Populate the domains used with the first candidates module list
    const map:string[] = []
    candidate_list.value[0].modules.forEach(module => {
        const domain = domain_list.value.find(item => item.content == module)
        map[domain.map] = domain.content
    })
    domains.value = map
})

// watches: state
// updates: process_list
watchEffect(() => {
    // Initialize process list after you clean the candidate list
    if(state.value == 'upload' || state.value == 'clean_candidate_list') return
    if(process_list.value.length) return
    process_list.value = candidate_list.value.slice()
})

// Encapsulating to hide helper functions
{
    const set_pass_expectations = (candidates:result_candidate[]) => {
        const scores = new Set(candidates.sort((a, b) => a.score > b.score ? -1 : 1).map(item => item.score))
        const score_pass_fail:{[score:string]:{decline:number, pass:number}} = {}
        let passed = candidates.filter(item => item.passed_karat).length
        let declined = 0
        scores.forEach(score => {
            const candidates_at_score = candidates.filter(candidate => candidate.score == score)
            declined += candidates_at_score.filter(item => !item.passed_karat).length

            score_pass_fail[score] = { pass: passed, decline: declined }
            passed -= candidates_at_score.filter(item => item.passed_karat).length
        })
        candidates.forEach(candidate => {
            candidate.pass_expectation = Math.floor(100 * score_pass_fail[candidate.score].pass / (score_pass_fail[candidate.score].pass + score_pass_fail[candidate.score].decline))
        })
    }

    const find_first_candidate = (passed:boolean, candidates:{score:number, scores:number[], passed_karat:boolean}[], i = -1) => {
        const default_score = passed ? -1 : 101
        const mod = passed ? 1 : -1
        candidates.sort((a, b) => {
            const a_score = (i == -1 ? a.score : a.scores[i]) * mod
            const b_score = (i == -1 ? b.score : b.scores[i]) * mod
            if(a_score == b_score) return a.passed_karat == passed ? -1 : 1
            return a_score < b_score ? -1 : 1
        })
        const found_index = candidates.findIndex(candidate => candidate.passed_karat == passed)
        const score = found_index > -1 ? (i == -1 ? candidates[found_index].score : candidates[found_index].scores[i]) : default_score
        const prev_score = found_index > 0 ? (i == -1 ? candidates[found_index - 1].score : candidates[found_index - 1].scores[i]) : default_score
        const count = found_index > -1 ? found_index : candidates.length
        return { score, count, prev_score }
    }

    const get_fit_count = (process_candidate_list:candidate[], weights:number[], weight_to_add:number, fit_memory:{[weight:number]:number}) => {
        if(fit_memory[weight_to_add] !== undefined) return fit_memory[weight_to_add]

        // Populate all the weights
        const all_weights = [...weights.map(weight => (100 - weight_to_add) * weight / 100), weight_to_add]

        // Adjust score using current weights
        process_candidate_list.forEach(item => item.score = all_weights.reduce((pv, cv, i) => pv + cv * item.scores[i] / 100, 0))

        // Get the fit count
        const fit_count = find_first_candidate(false, process_candidate_list).count + find_first_candidate(true, process_candidate_list).count
        
        // Return
        return fit_memory[weight_to_add] = fit_count
    }

    const process = (process_candidate_list:candidate[], at:number = 1, weights:number[] = [100]) => {
        const number_of_modules = process_list.value[0].modules.length
        if(at > 1) {
            let max_fit_weight:null|number = null
            const fit_memory:{[weight:number]:number} = {}
            const cur_fit_info:{weight:number, count:number}[] = []

            let low_weight = 0
            let hi_weight = 100
            while(hi_weight - low_weight > 2){
                cur_fit_info.length = 0

                const mod = (hi_weight - low_weight) / 3
                const mid1_weight = Math.floor(mod + low_weight)
                const mid2_weight = Math.floor(2 * mod + low_weight)

                cur_fit_info.push({weight: low_weight, count: get_fit_count(process_candidate_list, weights, low_weight, fit_memory)})
                cur_fit_info.push({weight: mid1_weight, count: get_fit_count(process_candidate_list, weights, mid1_weight, fit_memory)})
                cur_fit_info.push({weight: mid2_weight, count: get_fit_count(process_candidate_list, weights, mid2_weight, fit_memory)})
                cur_fit_info.push({weight: hi_weight, count: get_fit_count(process_candidate_list, weights, hi_weight, fit_memory)})
                if(max_fit_weight && !cur_fit_info.find(item => item.weight == max_fit_weight)) cur_fit_info.push({weight: max_fit_weight, count: get_fit_count(process_candidate_list, weights, max_fit_weight, fit_memory)})

                cur_fit_info.sort((a, b) => a.weight > b.weight ? 1 : -1)
                const max_count = cur_fit_info.reduce((pv, cv) => Math.max(pv, cv.count), 0)
                const max_fit_index = cur_fit_info.findIndex(item => item.count == max_count)

                max_fit_weight = cur_fit_info[max_fit_index].weight

                let low_index = max_fit_index
                let hi_index = max_fit_index
                while(low_index > 0 && cur_fit_info[low_index].count == max_count) low_index--
                while(hi_index < cur_fit_info.length - 1 && cur_fit_info[hi_index].count == max_count) hi_index++
                low_weight = cur_fit_info[low_index].weight
                hi_weight = cur_fit_info[hi_index].weight
                if(low_index == 0 && hi_index == cur_fit_info.length - 1){
                    if(low_weight == max_fit_weight) hi_weight--
                    else low_weight++
                }
            }
            
            const max_fit_count = get_fit_count(process_candidate_list, weights, max_fit_weight, fit_memory)
            const target_fit_weight = max_fit_count == get_fit_count(process_candidate_list, weights, 50, fit_memory) ? 50 : max_fit_count == fit_memory[100] ? 100 : max_fit_count == fit_memory[0] ? 0 : max_fit_count == get_fit_count(process_candidate_list, weights, 50, fit_memory) ? 50 : max_fit_weight

            console.log(fit_memory)
            debugger
            weights = [...weights.map(weight => (100 - target_fit_weight) * weight / 100), target_fit_weight]
        }
        if(at < number_of_modules) weights = process(process_candidate_list, at + 1, weights)
        return weights
    }

    // watches: ignore
    // watches: process_list
    // watches: results -> candidates
    // updates: results -> candidates -> score & system_rec
    watchEffect(() => {
        // Filter Data by Auto DNP
        const filtered_out_anomolus_candidate_ids = anomolus_candidates.value.filter(candidate => Math.abs(2 * candidate.pass_expectation - 100) > ignore.value).map(candidate => candidate.id)
        const process_candidates = results.value.candidates.filter(candidate => filtered_out_anomolus_candidate_ids.includes(candidate.id) == false)

        // Set candidate's scores
        results.value.candidates.forEach(candidate => candidate.score = candidate.scores.reduce((pv, cv, i) => pv == -1 || cv < results.value.auto_dnp[i] ? -1 : pv + cv * results.value.weights[i] / 100, 0))

        // Get DNP bar
        const first_pass = find_first_candidate(true, process_candidates)
        const dnp = first_pass.prev_score == -1 ? 0 : (first_pass.prev_score + first_pass.score) / 2
                    
        // Get ITNR bar
        const first_decline = find_first_candidate(false, process_candidates)
        const itnr = first_decline.prev_score == 101 ? 0 : (first_decline.prev_score + first_decline.score) / 2

        // Set candidate's recommendation
        results.value.candidates.forEach(candidate => candidate.system_rec = candidate.score < dnp ? 'DNP' : (candidate.score > itnr ? 'ITNR' : 'RFR'))

        set_pass_expectations(results.value.candidates)

        // Save results
        results.value.candidates.sort((a, b) => a.score > b.score ? -1 : 1)
        results.value.dnp = dnp
        results.value.itnr = itnr
    })

    // watches: process_list
    // updates: results
    watchEffect(() => {
        if(process_list.value.length == 0) return

        const ret:results = {
            weights: [],
            dnp: 0,
            itnr: 0,
            candidates: [],
            auto_dnp: []
        }

        const memory = {
            max_fit_count: 0,
            final_weights: []
        }

        // Get Aggressive Auto DNP levels
        const number_of_modules = process_list.value[0].modules.length
        const aggressive_auto_dnp:{index:number, level:number}[] = []
        for(let i = 0; i < number_of_modules; i++) aggressive_auto_dnp.push({
            index: i,
            level: find_first_candidate(true, process_list.value, i).prev_score + 1
        })

        // Get Auto DNP levels and filter candidates
        let filtered_process_list = process_list.value.slice()
        aggressive_auto_dnp.sort((a, b) => a.level < b.level ? -1 : 1).forEach(({index, level}) => {
            ret.auto_dnp[index] = find_first_candidate(true, filtered_process_list, index).prev_score + 1
            filtered_process_list = filtered_process_list.filter(candidate => candidate.scores[index] >= ret.auto_dnp[index])
        })

        // Process
        console.time(`Process`)
        ret.weights = process(filtered_process_list)
        console.timeEnd(`Process`)

        // Build out candidate data
        ret.candidates = [...candidate_list.value].map(candidate => {
            const c:result_candidate = {
                id: candidate.id,
                name: candidate.candidate,
                system_rec: 'RFR',
                client_rec: candidate.client_rec,
                score: candidate.scores.reduce((pv, cv, i) => pv + cv * ret.weights[i] / 100, 0),
                scores: candidate.scores,
                pass_expectation: 0,
                passed_karat: candidate.passed_karat
            }
            return c
        })

        filtered_process_list.forEach(candidate => candidate.score = candidate.scores.reduce((pv, cv, i) => pv + cv * ret.weights[i] / 100, 0))
        
        // Get DNP bar
        const first_pass = find_first_candidate(true, filtered_process_list)
        const dnp = first_pass.prev_score == -1 ? -1 : (first_pass.prev_score + first_pass.score) / 2
        
        // Get ITNR bar
        const first_decline = find_first_candidate(false, filtered_process_list)
        const itnr = first_decline.prev_score == 101 ? -1 : (first_decline.prev_score + first_decline.score) / 2

        // Get pass/decline fit score
        set_pass_expectations(ret.candidates)
        //const pass_score = ret.candidates.reduce((pv, cv) => pv + Math.pow(cv.pass_expectation - (cv.passed_karat ? 0 : 100), 2), 0)

        results.value = ret
    })
}

/*****************************************
 **************** METHODS ****************
 ****************************************/

function removeCandidate(id:number){
    const i = candidate_list.value.findIndex(candidate => candidate.id == id)
    if(i == -1) return
    candidate_list.value.splice(i, 1)
}

function removeContent(i){
    candidate_list.value = candidate_list.value.filter(item => item.modules.includes(domain_list.value[i].content) === false)
    domain_list.value.splice(i, 1)
}

function next(){
    const next_state_map = {
        upload: 'clean_candidate_list',
        clean_candidate_list: 'edit_domains',
        edit_domains: 'view_results'
    }
    state.value = next_state_map[state.value]
}

function copy_results(){
    let copy = 'Candidate,Issue,Confidence\n'
    for(const candidate of anomolus_candidates.value){
        copy += `"${candidate.name}","${candidate.passed_karat ? `Should have been declined` : `Should have passed`}",${Math.abs(2 * candidate.pass_expectation - 100)}%\n`
    }
    const type = 'text/plain'
    const blob = new Blob([copy], { type })
    let data = [new ClipboardItem({ [type]: blob })]
    navigator.clipboard.write(data)
}

const avg_scores = computed(() => {
    const dimensions = candidate_list.value[0]?.scores.length ?? 0
    const avg_scores = []
    for(let i = 0; i < dimensions; i++) avg_scores[i] = candidate_list.value.map(item => item.scores[i]).reduce((pv, cv) => pv + cv) / candidate_list.value.length
    return avg_scores
})

const normalize_weight = computed(() => {
    const min_score = Math.min(...avg_scores.value)
    const score_ratio = avg_scores.value.map(item => item / min_score)
    return results.value.weights.slice().map((weight, i) => weight * score_ratio[i])
})
</script>

<template lang="pug">
main
    .fixed(ref="header")
        .error(:class="{hide:!app_error}") {{app_error ? app_error : 'There is no error'}}
        .controls
            button(v-if="state != 'view_results'" @click="next" :disabled="next_button_disabled") Next
            div(v-if="state == 'view_results'") Ignore anomolies greater than
                input(type="range" min="49" max="100" v-model="ignore")
                span {{ignore}} 
        .headers 
            textarea(v-if="state == 'upload'" v-model.trim="raw_candidate_data")
            .candidates_header(v-if="state == 'clean_candidate_list'")
                h1 Content 
                div(class="grid" style="--grid-size:4")
                    template(v-for="domain, i of domain_list")
                        button(@click="removeContent(i)") X
                        span.content {{domain.content}}
                        span.count {{domain.count}}
                        select(v-model="domain.map")
                            option(v-for="n of max_modules_used" :value="n - 1") {{n}}
                br
                h1 Candidates
    .body(:style="{'margin-top':header_height}")
        div(v-if="state == 'clean_candidate_list'" class="grid" style="--grid-size:4")
            template(v-for="candidate of sorted_candidate_list" :key="candidate.id")
                button(@click="removeCandidate(candidate.id)") X
                span(:class="{flagged:candidate.flag}") {{candidate.candidate}}
                span {{candidate.passed_karat}}
                span {{candidate.scores}}
        .modules(v-if="state == 'edit_domains'") Domain{{domains.length > 1 ? 's' : ''}}
            input(v-for="_domain, i of domains.length" v-model="domains[i]")
        .results(v-if="state == 'view_results'")
            br
            h1 Alignment V3 Config
            h2(:class="alignment_confidence_class") Alignment Confidence {{Math.floor(1000 - 1000 * anomolus_candidates.length / results.candidates.length) / 10}}%
            div(class="grid" :style="`--grid-size:${domains.length + 1}`")
                span
                span(v-for="domain of domains" class="bold") {{domain}}
                span(class="bold") Weight
                span(v-for="weight of results.weights" class="center") {{weight}}
                span(class="bold") Auto DNP
                span(v-for="auto_dnp of results.auto_dnp" class="center") {{auto_dnp}}
            br
            div(class="center bold") ITNR Bar {{results.itnr.toFixed(2)}} --- DNP Bar {{results.dnp.toFixed(2)}}
            h1
                span Anomolus Candidates
                button(@click="copy_results") Copy For Spreasheet
            template(v-if="anomolus_candidates.length")
                div(class="grid" style="--grid-size:3")
                    span(class="bold") Candidate
                    span(class="bold") Issue 
                    span(class="bold") Confidence 
                    template(v-for="candidate of anomolus_candidates")
                        span(:class="{'color-light-grey': Math.abs(2 * candidate.pass_expectation - 100) > ignore}") {{candidate.name}}
                        span(:class="{'color-light-grey': Math.abs(2 * candidate.pass_expectation - 100) > ignore}") {{candidate.passed_karat ? `Should have been declined` : `Should have been brought onsite`}}
                        span(:class="{'color-light-grey': Math.abs(2 * candidate.pass_expectation - 100) > ignore}" class="center") {{Math.abs(2 * candidate.pass_expectation - 100)}}%
            div(v-else class="center") There are no anomolus candidates
            h1 Candidate Results
            div(class="grid" style="--grid-size:7")
                span(class="bold") Candidate
                span(class="bold") State 
                span(class="bold") V3 Rec
                span(class="bold") Rec Reviews
                span(class="bold") V3 Score
                span(class="bold") Raw Scores 
                span(class="bold") Pass Expectation
                template(v-for="candidate of results.candidates")
                    span {{candidate.name}}
                    span {{candidate.passed_karat ? 'Passed' : 'Declined'}}
                    span(class="center") {{candidate.system_rec}}
                    span(class="center") {{candidate.client_rec}}
                    span(class="center") {{candidate.score.toFixed(2)}}
                    span {{candidate.scores}}
                    span(class="center") {{candidate.pass_expectation}}%
</template>

<style>
h1, h2, .center {
    text-align: center;
}
h1 > span {
    padding-right: 16px;
}
h1 {
    display: flex;
    width:fit-content;
    margin-inline: auto;
}
.bold {
    font-weight: bold;
}
.grid {
    display: grid;
    grid-template-columns: repeat(var(--grid-size), auto);
    gap: 16px;
    width: fit-content;
    text-align: left;
    margin: auto;
}
.color-red {
    color:red;
}
.color-orange {
    color: orange;
}
.color-green {
    color: green;
}
.color-light-grey {
    color: lightgray;
}

html, body, #app, main {
    margin: 0%;
    height: 100%;
}

.modules input {
    margin-left: 8px;
}

.content_container .count {
    text-align: right;
}

.fixed {
    position:fixed;
    width: 100%;
    text-align: center;
    background-image: linear-gradient(to top, transparent, white 16px);
}

#app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
}

main {
    display: flex;
    flex-direction: column;
    align-items: center;
}

.hide {
    visibility: hidden;
}

.flagged {
    background-color: yellow;
}

.error {
    background-color: red;
    color: white;
    font-weight: bold;
    padding: 4px 8px;
    margin-top: 40px;
    margin-bottom: 20px;
}

.controls{
    text-align: center;
    margin-bottom: 20px;
}

textarea {
    width: 600px;
    height: 400px;
}
</style>