<template>

    <br>
    <div class="flex"><input v-model="seq_raw" :size="seq_raw.length" class="border text-center m-auto"/></div>
    <br>
    <div class="flex "><div class="flex-1">Длина: {{seq_raw.length}}</div> <div  class="flex-0">Значения: <input v-model="values_raw" :size="values_raw.length" class="border text-center"/></div></div>
    <table>
        <thead>
            <tr>
                <td class="p-1 border">Тест</td>
                <td class="p-1 border">В среднем</td>
                <td class="p-1 border">У вас</td>
                <td class="p-1 border">σ в среднем</td>
                <td class="p-1 border">σ у вас</td>
                <td class="p-1 border">Шанс на это</td>
            </tr>
        </thead>
        <tbody>
            <tr v-for="(test, key) in tests">
                <td class="p-1 border" >{{test.title}}<button @click="help=key">❓</button>️</td>
                <td class="p-1 border">{{probabilities[key].mean.length === 1 ? Math.round(probabilities[key].mean[0]*100)/100: ''}}</td>
                <td class="p-1 border">{{probabilities[key].your.length === 1 ? Math.round(probabilities[key].your[0]*100)/100: ''}}</td>
                <td class="p-1 border">{{Math.round(probabilities[key].median*100)/100}}</td>
                <td class="p-1 border">{{Math.round(probabilities[key].your_dis*100)/100}}</td>
                <td class="p-1 border">
                    <template v-if="probabilities[key].prob > 0.1">🙂</template>
                    <template v-if="probabilities[key].prob <= 0.1 && probabilities[key].prob > 0.01">🤨</template>
                    <template v-if="probabilities[key].prob < 0.01">😡</template>

                    {{Math.round(probabilities[key].prob*10000)/100}}%
                </td>
            </tr>
        </tbody>
    </table>
    <br>
    <div class="flex border border-dashed p-5" v-if="help">
        {{tests[help].description}}
    </div>


</template>

<script setup>

let tests_count = 10000

import {computed, ref} from "vue";

const seq_raw = ref('010101')
const values_raw = ref('01')
const help = ref(null)
const probability = ref(1)

let tests = {
    test1: {
        title: 'Cумма',
        description: 'Тест оценивает сумму членов ряда и сравнивает со среднестатистической',
        code: seq => [seq.reduce((a,b) => a+b)]
    },
    test3: {
        title: 'Число перемен',
        description: 'Тест оценивает число перемен (то есть изменений числа) и сравнивает со среднестатистическим',
        code: seq => {
            let jumps = 0
            let current = -1
            seq.forEach(s => {
                if(s !== current){
                    jumps += 1
                    current = s
                }
            })
            return [jumps]
        }
    },
    test4: {
        title: 'Однородность',
        description: 'Тест находит самую длинную последовательность одинаковых чисел и сравнивает со среднестатистической',
        code: seq => {
            let max = 0
            let current = -1
            let length = 0
            seq.forEach(s => {
                if(s === current){
                    length++
                }else{
                    if(max < length)
                        max = length
                    current = s
                    length = 1
                }
            })
            return [max]
        }
    }
}

/** @type {{value:Number[]}} */
const seq = computed(() => {
    let seq_lit = []
    for (let i = 0; i < seq_raw.value.length; i++) {
        let s = seq_raw.value[i]
        if (s.match(/([0-9])/g) && values_raw.value.includes(s)){
            seq_lit.push(s)
        }
    }
    seq_raw.value = seq_lit.join('')

    if(seq_lit.length === 0)
        return [0]

    return seq_lit.map(i => Number(i))
})

/** @type {{value:Number[]}} */
const values = computed(() => {
    let values_lit = []
    for (let i = 0; i < values_raw.value.length; i++) {
        let s = values_raw.value[i]
        if (s.match(/([0-9])/g)){
            values_lit.push(s)
        }
    }
    values_lit = [...new Set(values_lit)]
    if(values_lit.length === 0)
        return [0]
    values_raw.value = values_lit.join(',')
    return values_lit.map(i => Number(i))
})


/** @param {Number} n
 * @param {Number[]} values
 * @return {Number[]} */
function get_random_seq(n, values){
    let seq = []
    let k = values.length

    for(let i=0; i < n; i++){
        seq.push(values[Math.floor(Math.random() * k)])
    }

    return seq
}

const probabilities = computed(() => {
    let results = {}

    const len = seq.value.length

    const samples = []

    for(let i=0; i < tests_count; i++) {
        samples.push(get_random_seq(len, values.value))
    }

    for (let key in tests) {

        let test = tests[key].code

        let samples_results = samples.map(test)


        let dem = samples_results[0].length
        let mean = []

        for(let i=0;i<dem;i++){
            mean.push(0)
        }
        samples_results.forEach((res) => {
            for(let i=0;i<dem;i++){
                mean[i] += res[i]
            }
        })
        for(let i=0;i<dem;i++){
            mean[i] = mean[i] / samples_results.length
        }


        const deviations = []
        samples_results.forEach((res) => {
            let deviation = 0
            for(let i=0;i<dem;i++){
                deviation += (res[i]-mean[i])**2
            }
            deviations.push(deviation)
        })


        const distribution = deviations.sort((a,b)=>{
            if(Number(a) > Number(b)) return 1
            if(Number(a) === Number(b)) return 0
            if(Number(a) < Number(b)) return -1
        })

        let value = test(seq.value)
        let Xi = 0
        for(let i=0;i<dem;i++){
            Xi += (value[i]-mean[i])**2
        }

        let left = 0

        for(let i=0; i<tests_count; i++){
            if(distribution[i] >= Xi) break
            left++
        }

        results[key] = {
            mean: mean,
            median: Math.sqrt(distribution[tests_count/2]),
            your: value,
            your_dis: Math.sqrt(Xi),
            prob: 1-left/tests_count
        }
    }
    return results
})




</script>
