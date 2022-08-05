<script>
const Ben = {
    props: {
        theme: {
            type: String,
            default: 'brooklyn',
            validator: (value) => {
                const allowedThemes = ['brooklyn', 'bronx', 'chinatown']
                return allowedThemes.includes(value)
            },
        },
        questionsURL: {
            type: String,
            default: null,
        },
        questionsArray: '',
    },
    data() {
        return {
            isMobile: false,
            w: window.innerWidth,
            h: window.innerHeight,
            orientation: null,
            wrong: 0,
            right: 0,
            unanswered: 0,
            inTransition: false,
            count: 0,
            loaded: false,
            questions: [],
            defaultQuestions: [
                {
                    questionText:
                        'If $x = -1$, what does $(x+1)^2$ evaluate to?',
                    choices: [0, 1, 2, 3],
                    answer: 0,
                    tags: ['sample'],
                },
            ],
            questionText: null,
            hintText: null,
            choices: [],
            values: [],
            answer: null,
            tags: [],
            currentQuestion: {},
            currentlySelectedChoice: null,
            blurb_app: `For parents and kids to try out.`,
            blurb_ben: `A NYC dad with kids at Hunter college HS and Brooklyn Tech.`,
            blurb_version:
                '9b6ca21 — Last updated August 4, 2022. More updates to come.',
        }
    },
    computed: {
        currentQuestionNumber: function () {
            if (this.questions && Array.isArray(this.questions)) {
                return this.questions.indexOf(this.currentQuestion) + 1
            }
            return -1
        },
        totalQuestions: function () {
            return this.questions.length || 0
        },
        questionsRemaining: function () {
            return this.totalQuestions >= this.currentQuestionNumber
        },
    },
    mounted() {
        // Detect if
        this.detectOrientation()
        window.addEventListener('resize', this.detectOrientation)
        // This event listener breaks iphone
        // screen.orientation.addEventListener('change', this.detectOrientation)
        let [jq, bs, bs_with_popper, popper, math1, math2, ax] = [
            'https://code.jquery.com/jquery-3.6.0.min.js',
            'https://cdn.jsdelivr.net/npm/bootstrap@5.2.0/dist/css/bootstrap.min.css',
            'https://cdn.jsdelivr.net/npm/bootstrap@5.2.0/dist/js/bootstrap.bundle.min.js',
            'https://unpkg.com/@popperjs/core@2',
            'https://polyfill.io/v3/polyfill.min.js?features=es6',
            'https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js',
        ]

        // Mathjax 1
        let fn_math = () => {
            // call the renderMath method of the component
            this.renderMath()
        }

        // jQuery
        if (typeof jQuery !== 'function') {
            this.insertDependency({
                src: jq,
            })
        }

        // Popper Function
        const fn_popper = () => {
            const tooltipTriggerList = document.querySelectorAll(
                '[data-bs-toggle="tooltip"]'
            )
            const tooltipList = [...tooltipTriggerList].map(
                (tooltipTriggerEl) => new bootstrap.Tooltip(tooltipTriggerEl)
            )
        }
        setTimeout(fn_popper, 500)

        // Bootstrap + Popper
        this.insertDependency({ src: bs_with_popper, id: 'bs' })

        // Bootstrap CSS
        this.insertDependency({
            type: 'text/css',
            src: bs,
            id: 'bss',
        })

        // Masonry
        // this.insertDependency({
        //     src: 'https://cdn.jsdelivr.net/npm/masonry-layout@4.2.2/dist/masonry.pkgd.min.js',
        // })

        // MathJax
        this.insertDependency({
            src: math1,
        })
        this.insertDependency({
            src: math2,
            id: 'MathJax-script',
            fn: fn_math,
        })
        // Axios
        const fn_axios = () => {
            this.loadQuestions()
        }
        this.insertDependency({
            src: 'https://cdnjs.cloudflare.com/ajax/libs/axios/0.27.2/axios.min.js',
            id: 'axios',
            integrity:
                'sha512-odNmoc1XJy5x1TMVMdC7EMs3IVdItLPlCeL5vSUPN2llYKMJ2eByTTAIiiuqLg+GdNr9hF6z81p27DArRFKT7A==',
            fn: fn_axios,
        })
    },
    watch: {
        questions: function (a, b) {
            // Always when a change occurs set to false
            if (a !== b) {
                this.loaded = false
            }
            // If the new value is legt, set loaded to truew
            if (a && Array.isArray(a) && a.length > 0) {
                this.loaded = true
                this.unanswered = this.questions.length
            } else {
                this.loaded = false
            }
        },
        loaded: function (a, b) {
            // If it changes to true from false

            if (a && a !== b) {
                let q = this.questions[0]
                this.displayQuestion(0)
            }
        },
        wrong: function (a, b) {
            this.unanswered = this.totalQuestions - (this.wrong + this.right)
        },
        right: function (a, b) {
            this.unanswered = this.totalQuestions - (this.wrong + this.right)
        },
    },
    methods: {
        fade(idx = 0) {
            let {
                questionText = null,
                choices = [],
                values = [],
                answer = null,
                tags = [],
                hint = null,
            } = this.questions[idx]

            // get 'this'
            let THIS = this
            // First get the target(s)
            const targets = '[data-role="question"]'

            // we fade out the currentQuestion
            let fadeOut = anime({
                targets: targets,
                opacity: {
                    value: 0,
                    duration: 300,
                    easing: 'easeInOutQuad',
                },
                complete: () => {
                    THIS.choices = this.formatChoices(choices)
                    THIS.values = this.forceArray(choices)
                    THIS.tags = this.formatTags(tags)
                    THIS.answer = answer
                    THIS.questionText = questionText
                    THIS.hintText = hint
                    THIS.currentQuestion = this.questions[idx]
                },
            })
            fadeOut.finished.then(() => {
                // Render the equations
                THIS.renderMath()
                // Now fade the new quetion IN
                let fadein = anime({
                    targets: targets,
                    opacity: {
                        value: 1,
                        duration: 250,
                        easing: 'easeInOutQuad',
                    },
                })
            })
        },
        detectOrientation() {
            // Detect and set orientation
            let o
            try {
                o =
                    (screen.orientation || {}).type ||
                    screen.mozOrientation ||
                    screen.msOrientation
            } catch (err) {
                o = ''
            }

            o = undefined === o ? '' : o

            this.orientation = o

            // Detect and set isMobile, width, and height vars
            let r = new RegExp(
                /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i
            )
            // Set values
            this.isMobile = r.test(navigator.userAgent)
            this.w = window.outerWidth
            this.h = window.outerHeight
            console.log(
                `w=${this.w}, h = ${this.h}, isMobile=${this.isMobile}, orientation=${this.orientation}`
            )
            return true
        },
        getBlurb(b) {
            switch (b) {
                case 'ben':
                case 'author':
                    // about me
                    return {
                        link: 'https://www.ben-willenbring.com/about',
                        html: `${this.blurb_ben} Click to learn more.`,
                    }
                    break

                case 'app':
                case 'demo':
                    // app
                    return {
                        link: 'https://www.ben-willenbring.com/shsat-back-story',
                        html: `${this.blurb_app} Click to learn more.`,
                    }
                    break
            }
        },
        enableTooltips() {
            // Code here
            const tooltipTriggerList = document.querySelectorAll(
                '[data-bs-toggle="tooltip"]'
            )
            const tooltipList = [...tooltipTriggerList].map(
                (tooltipTriggerEl) => new bootstrap.Tooltip(tooltipTriggerEl)
            )
            console.log('popper!!!!!')
        },
        getQuestionsFromParams(param = 'quiz') {
            try {
                // This method parses the querystring
                let jsonFile = new URLSearchParams(window.location.search).get(
                    param
                )
                // It's being pulled from a remote url or locally
                if (jsonFile.startsWith('http') || jsonFile.endsWith('.json')) {
                    jsonFile = jsonFile
                } else {
                    jsonFile = `${jsonFile}.json`
                }

                return jsonFile
            } catch (err) {
                return null
            }

            return null
        },
        forceArray(val) {
            if (Array.isArray(val)) return val

            try {
                return eval(val)
            } catch (err) {
                return []
            }
        },
        formatChoices(choices) {
            let formattedChoices = []
            // we don't know if choices = string, array, or something else
            if (!choices) {
                return formattedChoices
            }
            if (typeof choices === 'string') {
                formattedChoices = this.forceArray(choices)
                // return formattedChoices
                // Format each choice
                formattedChoices = formattedChoices.map((c) =>
                    this.formatChoice(c)
                )
                return formattedChoices
            } else if (Array.isArray(choices)) {
                // Format each choice
                formattedChoices = choices.map((c) => this.formatChoice(c))
                return formattedChoices
            } else {
                return formattedChoices
            }
        },
        formatChoice(val) {
            // val could either be string or number
            if (typeof val === 'string') {
                return val
            } else if (typeof val === 'number') {
                return `$${val}$`
            } else {
                return ''
            }
        },
        formatTags(tags) {
            let defaultTags = []
            if (!tags || !Array.isArray(tags)) {
                return defaultTags
            }
            if (Array.isArray(tags)) {
                try {
                    return tags.map((tag) => this.formatTag(tag))
                } catch (err) {
                    return defaultTags
                }
            }

            return defaultTags
        },
        formatTag(tag) {
            if (tag && typeof tag === 'string') {
                return tag.toLowerCase()
            } else if (typeof tag === 'object' && tag.name) {
                return tag.name.toLowerCase()
            } else {
                return ''
            }
            return ''
        },
        increment(arg = 1) {
            this.count++
        },
        testMethod(p) {
            // $('button[data-bs-slide-to="1"]').trigger('click');
            alert('wow' + p)
        },
        toggleQuestion(idx) {
            // Takes an index
            this.displayQuestion(idx)
        },
        renderMath(attempts = 1) {
            attempts++
            MathJax.typesetPromise().then((r) => console.log(r))
        },
        displayQuestion(idx) {
            if (idx >= 0 && idx < this.questions.length) {
                // This fades stuff OUT, then IN
                this.fade(idx)
            }
        },
        async loadQuestions() {
            /**
             * 3 Scenarios
             *    1. questions are passed as params. Example...
             *      {base url}?questions=questions=[{"questionText": "Ben"}]
             *
             *    2. questions are passed in as a string via props...
             *      let q = JSON.stringify([{"questionText": "Ben"}])
             *      template: `<shsat questionsArray='${q}'></shsat>`
             *
             *    3. questions are passed in as a string to an URL
             *      template: `<shsat questionsURL='${questionsURL}'></shsat>`
             */
            let questionsAsParams = this.getQuestionsFromParams()
            if (this.questionsURL || questionsAsParams) {
                // Favor querystring
                let q_url = questionsAsParams
                    ? questionsAsParams
                    : this.questionsURL
                q_url = `${q_url}?t=${new Date().getTime()}`

                const sep = '❤️ '.repeat(25)
                this.log('Loading questions...!')

                try {
                    const r = await axios.get(`${q_url}`)

                    // Questions are presumed to be... { "questions": [ ... an array of questions... ] }
                    const questions =
                        r.data && r.data.questions ? r.data.questions : []

                    if (questions && questions.length > 0) {
                        this.setQuestions(questions)
                    }
                } catch (err) {
                    this.log(err)
                }
            } else if (this.questionsArray) {
                // questionsArray is a prop passed in as a stringified Array of questions
                let questionsArray =
                    typeof this.questionsArray === 'string'
                        ? eval(this.questionsArray)
                        : this.questionsArray

                this.setQuestions(questionsArray)
            } else {
                this.log(`😢 Using default questions!`)
                // Use the default questions
                let d = JSON.parse(JSON.stringify(this.defaultQuestions))
                this.setQuestions(d)
            }
        },
        log(msg) {
            let sep = '❤️ '.repeat(25)
            console.log(sep)
            console.log(msg)
            console.log(sep)
        },
        setQuestions(q = []) {
            // This sets the questions based on the axios request's response
            if (q && Array.isArray(q) && q.length > 0) {
                // Set the component questions correctly
                this.questions = q.map((item) => {
                    // Give one more prop
                    item.idx = q.indexOf(item)
                    item.values = this.forceArray(item.choices)
                    // This stores what the user selects
                    item.userChoice = null
                    item.status = 'unanswered'
                    item.isCorrect = null
                    return item
                })
            }
        },
        insertDependency({
            type = 'text/javascript',
            src = '',
            id = '',
            fn = null,
        } = {}) {
            // Two allowed types
            const allowedTypes = ['text/javascript', 'text/css']
            // Initialize script and tag
            let [script, tag] = [null, 'script']

            if (
                !src &&
                type.includes('javascript') &&
                fn &&
                typeof fn === 'object'
            ) {
                fn()
            } else if (
                src &&
                src.startsWith('http') &&
                allowedTypes.includes(type)
            ) {
                if (!type.includes('javascript')) {
                    tag = 'link'
                }
                script = document.createElement(tag)
                if (id) script.id = id

                if (type.includes('javascript')) {
                    // js
                    script.type = type
                    script.async = true
                    script.src = src
                } else {
                    // css
                    script.rel = 'stylesheet'
                    script.crossorigin = 'anonymous'
                    script.href = src
                }
                // Inject into dom and conditionally execute a callback
                document.getElementsByTagName('head')[0].appendChild(script)
                if (fn && typeof fn === 'function') {
                    script.onload = () => fn()
                }
            }
        },
        getCircleClass(idx = 0) {
            // compare idx to currentQuestion
            if (idx === this.currentQuestionNumber - 1) {
                return 'small badge rounded-pill text-bg-success selected'
            } else {
                return 'small badge rounded-pill text-bg-secondary'
            }
        },
        selectChoice(userChoice) {
            // What's the currentQuestion
            let q = this.currentQuestion
            // Persist the user's selection for the current question
            this.setSelectionForQuestion(q.idx, userChoice)
            let { choices, answer, idx } = q

            this.currentQuestion.isCorrect =
                String(userChoice) == String(answer) ? true : false

            this.calculateScore()
        },
        calculateScore() {
            let right = this.questions.filter(
                (item) => item.isCorrect == true
            ).length

            let wrong = this.questions.filter(
                (item) => item.isCorrect == false
            ).length

            this.wrong = wrong
            this.right = right
        },
        setSelectionForQuestion(idx, userChoice) {
            this.questions[idx].userChoice = userChoice
            this.questions[idx].status = 'answered'
        },
        getSelectionForQuestion(idx) {
            try {
                return this.questions[idx].userChoice || null
            } catch (err) {
                return null
            }
        },
        getThemeStyles() {
            if (!this.theme) return ''
            // These are the background image urls
            let urls = {
                brooklyn:
                    'https://images.squarespace-cdn.com/content/v1/5dabaee11e9d9809a8816556/d591a691-11b3-46e2-963e-8bc94dd25ddb/ke17ZwdGBToddI8pDm48kOyeI2mcCTajW7wMXBwmmj0UqsxRUqqbr1mOJYKfIPR7LoDQ9mXPOjoJoqy81S2I8N_N4V1vUb5AoIIIbLZhVYy7Mythp_T-mtop-vrsUOmeInPi9iDjx9w8K4ZfjXt2dp-NHKnzrD7M1S1eExpO4HnV308_ZVFASBb2nRCc3RLGP7cJNZlDXbgJNE9ef52e8w/theme-biggie-bw.jpg',
                bronx: 'https://images.squarespace-cdn.com/content/v1/5dabaee11e9d9809a8816556/1621203066818-GOBFWTGXRUVP2VRSLZHG/ke17ZwdGBToddI8pDm48kOyeI2mcCTajW7wMXBwmmj0UqsxRUqqbr1mOJYKfIPR7LoDQ9mXPOjoJoqy81S2I8N_N4V1vUb5AoIIIbLZhVYy7Mythp_T-mtop-vrsUOmeInPi9iDjx9w8K4ZfjXt2dp-NHKnzrD7M1S1eExpO4HnV308_ZVFASBb2nRCc3RLGP7cJNZlDXbgJNE9ef52e8w/theme-bronx.jpg',
                chinatown:
                    'https://images.squarespace-cdn.com/content/v1/5dabaee11e9d9809a8816556/1621810540000-9E8KGAUGA9KI8BKZDIOY/ke17ZwdGBToddI8pDm48kOyeI2mcCTajW7wMXBwmmj0UqsxRUqqbr1mOJYKfIPR7LoDQ9mXPOjoJoqy81S2I8N_N4V1vUb5AoIIIbLZhVYy7Mythp_T-mtop-vrsUOmeInPi9iDjx9w8K4ZfjXt2dp-NHKnzrD7M1S1eExpO4HnV308_ZVFASBb2nRCc3RLGP7cJNZlDXbgJNE9ef52e8w/theme-chinatown.jpg',
            }
            // Gradient color inflection points
            let gradient = [
                'rgba(255, 255, 255, 0.15) 0%',
                'rgba(255, 255, 255, 0.95) 20%',
                'rgba(255, 255, 255, 0.95) 70%',
                'rgba(255, 255, 255, 0.25) 90%',
                'rgba(255, 255, 255, 0.15) 100%',
            ]
            // Return a string
            return `background-image: linear-gradient(to bottom, ${gradient.join(
                ', '
            )}), url(${urls[this.theme]})`
        },
        getClassFor({ el = null, params = {} } = {}) {
            switch (el) {
                case 'choice':
                    let idx = this.currentQuestion.idx
                    let allAnswers = this.questions.filter(
                        (item) => item.userChoice
                    )

                    let val = params.value || null
                    // If no attempt has been made, return 'choice'
                    try {
                        if (
                            val &&
                            String(val) ==
                                String(this.questions[idx].userChoice)
                        ) {
                            return 'choice selected'
                        } else {
                            return 'choice'
                        }
                    } catch (err) {
                        this.log('error...')
                        this.log(err)
                        return 'choice'
                    }
                    break
                case 'circleContainer':
                    const answeredAlready =
                        this.getSelectionForQuestion(params.idx) || false
                    // Is it the current and
                    if (
                        answeredAlready &&
                        params.idx != this.currentQuestion.idx
                    ) {
                        return ['d-inline-block', 'answered']
                    } else if (
                        answeredAlready &&
                        params.idx == this.currentQuestion.idx
                    ) {
                        return ['d-inline-block', 'selected', 'answered']
                    } else if (params.idx == this.currentQuestion.idx) {
                        return ['d-inline-block', 'selected']
                    } else {
                        return 'd-inline-block'
                    }

                    break

                default:
                    break
            }
        },
    },
}

export default Ben
</script>

<template ref="foo">
    <!-- outer shell -->
    <div class="d-flex justify-content-center">
        <div
            id="outerShell"
            v-show="loaded"
            :style="getThemeStyles()"
            :class="`position-relative container-fluid m-1 border border-4 border-secondary theme`"
        >
            <!-- Fader -->
            <div data-role="fader"></div>
            <!-- Begin Table -->
            <div class="table-container">
                <table class="question-table rounded-2 w-100 p-0 m-0">
                    <tr>
                        <td class="verticalSpacer" style="">&nbsp;</td>
                        <td valign="top">
                            <!-- Question Container  -->
                            <div
                                data-role="question-container"
                                class="container-fluid p-0"
                            >
                                <div
                                    class="d-flex-row"
                                    data-role="question"
                                    :data-test="currentQuestionNumber"
                                >
                                    <!-- Question Text (allow setting html) -->
                                    <div
                                        class="h4"
                                        data-role="questionText"
                                        v-html="questionText"
                                    ></div>

                                    <!-- Choices -->
                                    <div
                                        data-role="choices"
                                        class="container-fluid"
                                    >
                                        <div>
                                            <ol
                                                class="choices"
                                                v-if="
                                                    choices.length &&
                                                    choices.length > 0
                                                "
                                            >
                                                <li
                                                    v-for="c in choices"
                                                    :key="choices.indexOf(c)"
                                                    @click="
                                                        selectChoice(
                                                            currentQuestion
                                                                .values[
                                                                choices.indexOf(
                                                                    c
                                                                )
                                                            ]
                                                        )
                                                    "
                                                    :class="
                                                        getClassFor({
                                                            el: 'choice',
                                                            params: {
                                                                value: currentQuestion
                                                                    .values[
                                                                    choices.indexOf(
                                                                        c
                                                                    )
                                                                ],
                                                            },
                                                        })
                                                    "
                                                >
                                                    <span
                                                        :data-question-idx="
                                                            currentQuestionNumber
                                                        "
                                                        :data-choice-idx="
                                                            choices.indexOf(c)
                                                        "
                                                        :data-value="
                                                            currentQuestion
                                                                .values[
                                                                choices.indexOf(
                                                                    c
                                                                )
                                                            ]
                                                        "
                                                        class="choice"
                                                        >{{ c }}</span
                                                    >
                                                </li>
                                            </ol>
                                        </div>
                                    </div>

                                    <!-- Hint & Tags -->
                                    <div
                                        style="border: 0px solid blue"
                                        data-role="hintsAndTags"
                                        class="position-relative"
                                    >
                                        <!-- Hint Container -->
                                        <div
                                            data-role="hint-container"
                                            v-if="hintText"
                                            class="d-inline-block"
                                        >
                                            <span>❤️ <b>Hint:</b></span>
                                            <span
                                                class="ms-1"
                                                v-html="hintText"
                                            ></span>
                                        </div>

                                        <!-- Tag Container -->
                                        <div
                                            data-role="tag-container"
                                            class="position-absolute w-100"
                                        >
                                            <div
                                                class="d-flex justify-content-end"
                                            >
                                                <div
                                                    class="d-inline-block"
                                                    v-for="tag of tags"
                                                    :key="tags.indexOf(tag)"
                                                >
                                                    <span
                                                        class="badge text-bg-ben me-2"
                                                        >{{ tag }}</span
                                                    >
                                                </div>
                                            </div>
                                        </div>
                                    </div>
                                </div>
                            </div>
                            <!-- End [data-type="question-container"] -->
                        </td>
                        <td class="verticalSpacer">&nbsp;</td>
                    </tr>

                    <!-- Row 2 -->
                    <tr>
                        <td colspan="3">
                            <!-- Put the hints and tags here -->
                            <div class="container-fluid">
                                <!-- Next / Previous buttons -->
                                <div
                                    v-if="totalQuestions > 0"
                                    data-role="question-buttons"
                                    class="container-fluid position-relative p-0 m-0 border border-0 border-warning bg-light"
                                >
                                    <div
                                        data-role="buttons"
                                        class="d-flex p-2 container-fluid justify-content-end border border-0 border-primary"
                                        style=""
                                    >
                                        <!-- Circles -->
                                        <div
                                            :class="w < 650 ? 'd-none' : 'col'"
                                        >
                                            <div
                                                data-role="circleContainer"
                                                v-for="q in questions"
                                                :key="questions.indexOf(q)"
                                                @click="
                                                    toggleQuestion(
                                                        questions.indexOf(q)
                                                    )
                                                "
                                                :class="
                                                    getClassFor({
                                                        el: 'circleContainer',
                                                        params: {
                                                            idx: questions.indexOf(
                                                                q
                                                            ),
                                                        },
                                                    })
                                                "
                                            >
                                                <div
                                                    :class="
                                                        getCircleClass(
                                                            questions.indexOf(q)
                                                        )
                                                    "
                                                    data-role="circle"
                                                >
                                                    &nbsp;
                                                </div>
                                            </div>
                                        </div>
                                        <!-- End Circles -->

                                        <!-- Previous & Next button -->
                                        <div
                                            data-role="button-container"
                                            class="p-0"
                                        >
                                            <!-- score board 
                            -->
                                            <div
                                                data-role="board"
                                                class="d-inline-block"
                                            >
                                                <div
                                                    class="rounded-1 me-1 small p-1"
                                                >
                                                    <!-- <span class="me-2">Progress:</span> -->
                                                    <div
                                                        class="badge badge-pill text-bg-danger me-1"
                                                        data-bs-toggle="tooltip"
                                                        data-bs-delay='{ "show": 0, "hide": 0 }'
                                                        data-bs-title="Total incorrectly answered questions"
                                                    >
                                                        {{ wrong }}
                                                    </div>
                                                    <div
                                                        class="badge badge-pill text-bg-success me-1"
                                                        data-bs-toggle="tooltip"
                                                        data-bs-title="Total correctly answered questions"
                                                    >
                                                        {{ right }}
                                                    </div>
                                                    <div
                                                        class="badge badge-pill text-bg-warning"
                                                        data-bs-toggle="tooltip"
                                                        data-bs-title="Total unanswered questions"
                                                    >
                                                        {{ unanswered }}
                                                    </div>
                                                </div>
                                            </div>

                                            <!-- Previous btn -->
                                            <button
                                                :disabled="
                                                    currentQuestionNumber === 1
                                                "
                                                @click="
                                                    toggleQuestion(
                                                        currentQuestionNumber -
                                                            2
                                                    )
                                                "
                                                type="button"
                                                class="btn btn-sm btn-primary border border-1 border-secondary me-1"
                                            >
                                                Prev
                                            </button>

                                            <!-- Next btn -->
                                            <button
                                                v-if="questionsRemaining"
                                                :disabled="
                                                    currentQuestionNumber >=
                                                    questions.length
                                                "
                                                @click="
                                                    toggleQuestion(
                                                        currentQuestionNumber
                                                    )
                                                "
                                                type="button"
                                                class="btn btn-sm btn-primary border border-1 border-secondary"
                                            >
                                                Next
                                            </button>
                                        </div>

                                        <!-- TODO: FIX UP-->
                                        <span class="me-2 d-none">
                                            You're on question #
                                            {{ currentQuestionNumber }}
                                        </span>
                                    </div>
                                </div>
                            </div>
                        </td>
                    </tr>

                    <!-- Row 3 -->
                </table>
            </div>

            <div class="container-fluid p-2">
                <!-- Branding / Tagline -->
                <div
                    data-role="branding"
                    class="container-fluid d-flex justify-content-end"
                >
                    <div
                        class="branding p-2 small"
                        style="font-weight: 500; font-size: 11px"
                    >
                        <!-- Version info -->
                        <a :href="getBlurb('app').link" target="_blank">
                            <div
                                data-bs-toggle="tooltip"
                                data-bs-html="true"
                                data-bs-delay='{"show":0,"hide":0}'
                                :data-bs-title="blurb_version"
                                class="d-inline-block pe-2 border border-1 border-top-0 border-bottom-0 border-start-0 border-secondary"
                            >
                                Version info
                            </div>
                        </a>

                        <!-- About the app -->
                        <a :href="getBlurb('app').link" target="_blank">
                            <div
                                data-bs-toggle="tooltip"
                                data-bs-html="true"
                                data-bs-custom-class="wide"
                                data-bs-delay='{"show":0,"hide":0}'
                                :data-bs-title="getBlurb('app').html"
                                class="d-inline-block ps-2 pe-2 border border-1 border-top-0 border-bottom-0 border-start-0 border-secondary"
                            >
                                About this demo
                            </div>
                        </a>

                        <!-- About ben -->
                        <a :href="getBlurb('ben').link" target="_blank">
                            <div
                                data-bs-toggle="tooltip"
                                data-bs-html="true"
                                data-bs-custom-class="wide"
                                data-bs-delay='{"show":0,"hide":0}'
                                :data-bs-title="getBlurb('ben').html"
                                class="ps-2 d-inline-block"
                            >
                                About Ben
                            </div>
                        </a>
                    </div>
                </div>
            </div>
            <!-- End Table -->
        </div>
    </div>

    <!-- FOR PRINT -->

    <div
        data-role="print"
        class="row"
        data-masonry='{"percentPosition": true }'
        style="position: relative; border: 2px solid grey"
    >
        <!-- Iterate over each question -->
        <div v-for="q in questions" class="card" data-role="print-question">
            <div class="col-sm-6 col-lg-4 mb-4">
                <div class="card border border-1 border-secondary">
                    question
                </div>
            </div>
        </div>
    </div>
</template>

<style>
@media screen {
    [data-role='print'] {
        display: none !important;
    }
}

@media print {
    /** This ensures the screen stuff is not printed */
    #outerShell {
        display: none;
    }
}

#outerShell {
    align-self: center !important;
    border-radius: 10px;
    padding: 5px;
    position: relative;
    width: 100%;
}
.table-container {
    border-radius: 5px;
    border: 1px solid #999999;
    background-color: rgba(255, 255, 255, 0.85);
}
.question-table {
    padding-bottom: 40px;
}
[data-role='fader'] {
    position: absolute;
    top: 0px;
    display: none;
    width: 100%;
    height: 200px;
    background-color: rgba(255, 255, 255, 0);
    border: 2px solid red;
    z-index: 10;
}
[data-role='question-buttons'] {
    margin-bottom: 25px;
    border: 1px solid red !important;
}

[data-role='branding'] {
    top: calc(100% - 0px);
    z-index: 5;
}

.branding {
    background-color: rgba(0, 0, 0, 1);
    color: #eeeeee;
    border-radius: 3px;
    display: block;
}

[data-role='board'] {
    margin-top: 14px;
    margin-left: 32px;
}
.floater {
    border: 1px solid #999999;
}
[data-role='board'] .badge {
    min-width: 30px;
}

.fadedOut {
    visibility: hidden;
    opacity: 0;
    transition: visibility 0s 0.2s, opacity 0.2s linear;
}
.fadedIn {
    visibility: visible;
    opacity: 1;
    transition: opacity 0.2s linear;
}

[data-role='print'] {
    padding: 0px;
    margin: 0px;
}

[data-role='print-question'] {
    display: block;
    border: 1px solid #eeeeee;
    page-break-inside: avoid;
    margin-top: 20px;
}
[data-role='print-question']:nth-of-type(1) {
    margin-top: 0px;
}
/* --------------------------------------------------
Mathjax
-------------------------------------------------- */
mjx-container,
[jax='CHTML'] {
    color: #395b64;
    font-weight: bolder;
}
mjx-container[display='true'] {
    font-size: 1.3em !important;
}

/* The part you actually see */
.tooltip-inner {
    /* These style the inside of the tooltip (the contents) */
    padding: 7px !important;
    text-align: left !important;
    background-color: #607eaa !important;
    opacity: 1;
}

.tooltip .tooltip-arrow::before {
    /* These 5 lines are necessary for coloring the tooltip arrow #607eaa*/
    color: #607eaa !important;
    border-bottom-color: #607eaa !important;
    border-top-color: #607eaa !important;
    border-width: 6px 6px 0 !important;
    z-index: -1 !important;
}

/** Custom Tooltips with `data-bs-custom-class` like this:
    <div
        data-bs-toggle="tooltip"
        data-bs-custom-class="wide"
        data-bs-title="Here is a custom tooltip">Hi there!</div>

    Note: `.wide` is the outer element. So... 👇🏽
 */

/* Make the custom tooltip fully opaque */
.tooltip,
.wide {
    opacity: 1 !important;
}
/* Make the custom tooltip wide */
.tooltip .wide .tooltip-inner {
    min-width: 300px;
}
/* .tooltip wide bs-tooltip-auto fade show */

.tooltip-inner {
    min-width: 290px;
}
</style>

<style scoped type="text/css">
.header {
    /* border-bottom: 1px solid #eeeeee; */
    padding: 0px 0px 20px 0px;
}

/* --------------------------------------------------
Question & question text
-------------------------------------------------- */
.verticalSpacer {
    height: 300px;
    width: 10px !important;
    /* background-color: rgba(255, 255, 255, 0.85); */
}
[data-role='question-container'] {
    border-radius: 7px;

    /* min-height: 580px; */
    /* margin-bottom: 55px; */
}

[data-role='question'] {
    padding: 20px 0px 20px 0px;
    position: relative;
}
[data-role='questionText'] {
    font-size: 1.5em;
    font-family: adobe-garamond-pro;
}

[data-role='hint-container'] {
    top: -10px;
    z-index: 6;
    font-size: 0.8em;
    color: #999999;
    padding: 5px;
}

[data-role='tag-container'] {
    top: -24px;
    z-index: 5;
}
.questionContainer {
    border-radius: 10px;
}

/* --------------------------------------------------
Choices
-------------------------------------------------- */
[data-role='choices'] {
    margin-top: 30px;
    margin-bottom: 10px;
    padding: 10px;
    background-color: rgba(238, 238, 238, 0.5);
    border-radius: 7px;
    border: 1px solid #cccccc;
    /* padding-bottom: 10px; */
}

ol.choices,
ul.choices {
    list-style-type: upper-alpha;
    font-weight: bolder;
}
.choice {
    font-weight: normal;
    font-size: 1em;
}
/* selected choice */
.choice.selected {
    background-color: rgba(247, 218, 30, 0.25);
}
.choices li {
    padding: 5px;
    background-color: none;
    font-size: 1.2em;
    color: #666666;
}

.choices li:hover,
.choices li.selected {
    cursor: pointer;
    color: #395b64;
    background-color: rgba(247, 218, 30, 0.25);
}
.choices li.selected {
    opacity: 1;
}

[data-role='circleContainer'] {
    padding: 0px;
    padding-right: 3px;
    padding-left: 3px;
    margin-right: 1px;
    margin-bottom: 1px;
    vertical-align: middle;
    font-size: 0.7em;
    line-height: 1.4em;
    border: 1px solid rgba(177, 225, 255, 0);
}
[data-role='circleContainer'].selected {
    background-color: rgba(177, 225, 255, 1);
    border: 1px solid rgba(185, 177, 255, 0.95);
}
[data-role='circleContainer'].answered {
    background-color: rgba(177, 225, 255, 0.75);
    border: 1px solid rgba(177, 225, 255, 0.95);
}

[data-role='circleContainer']:hover {
    background-color: rgba(177, 225, 255, 0.5);
    cursor: pointer;
}

/* [data-role='circle']:hover {
    cursor: pointer;
} */
[data-role='circle'] {
    height: 5px;
    width: 5px;
    font-size: 0px;
    vertical-align: middle;
    text-align: center;
    opacity: 0.5;
}
[data-role='circle'].selected {
    opacity: 1;
}

/* --------------------------------------------------
Buttons
-------------------------------------------------- */
[data-role='button-container'] {
    vertical-align: top !important;
}

.text-bg-ben {
    background-color: rgba(206, 227, 227, 1);
    color: #222222;
}
.theme {
    background-color: #eeeeee;
    background-image: linear-gradient(
        to bottom,
        rgba(255, 255, 255, 0.15) 0%,
        rgba(255, 255, 255, 0.95) 20%,
        rgba(255, 255, 255, 0.95) 70%,
        rgba(255, 255, 255, 0.25) 90%,
        rgba(255, 255, 255, 0.15) 100%
    );
    background-size: cover;
    background-position-x: right;
    background-position-y: bottom;
}
</style>
