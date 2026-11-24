---
title: "Free Vue.js Course: Composition API, Pinia, Vue Router, Testing, SSR & Nuxt"
description: "Free Vue 3 course: 36 lessons, 3 projects and 1 capstone on the Composition API, reactivity internals, Vue Router, Pinia, testing, accessibility, SSR, Nuxt and production deployment."
---

# Vue.js Mastery Path

Vue is the framework people describe as *approachable*, and it is: you can add it to one
page with a script tag, or scaffold a full application in a minute. What makes it worth
studying in depth is that the approachable surface sits on a small, precise core — a
reactivity system built on JavaScript proxies, a template compiler that knows which parts
of your markup can change, and a renderer that uses that knowledge to do as little work as
possible. Once you understand those three pieces, most Vue behaviour stops being
surprising.

This course teaches **Vue 3 with the Composition API and `<script setup>`**, targeting
**Vue 3.5** with Vue Router 5, Pinia 4 and Vite 8 (the versions on npm while it was
written). Level 1 builds a habit tracker from an empty folder: tooling, templates,
`ref`/`reactive`/`computed`, watchers, props and emits, `v-model`, slots and transitions.
Level 2 turns components into applications: composables, provide/inject, routing, Pinia,
async data, form validation and TypeScript — ending with a routed recipe book. Level 3 goes
under the hood: reactivity and rendering internals, directives and plugins, the built-in
components, code splitting, performance, testing, accessibility and security — ending with
an accessible Kanban board. Level 4 is production: server-side rendering by hand and with
Nuxt, production builds, component libraries, architecture, Vue 2 migration, i18n,
deployment and observability, finishing with a server-rendered capstone.

Every lesson has a **How It Actually Works** section — what the compiler turns a template
into, why destructuring a reactive object silently breaks updates, how a `computed` knows
when to recompute, why `v-if` and `v-for` on one element fail in Vue 3, how SSR state
reaches the browser without an XSS hole.

!!! note "Verified against real builds — and honest about what can't be run"
    The code in this course was built with Vite, tested with Vitest and Vue Test Utils,
    type-checked with `vue-tsc`, and — for anything interactive or server-rendered — run in
    a real browser or against a real server while writing. Output shown on these pages is
    output we actually observed, including first attempts that failed, because the
    diagnosis is usually the most useful part. Where something couldn't be run locally
    (a Docker image, a hosting provider, a hosted error service), the lesson says so.

## How the program is organized

| Level | Focus | Modules |
|-------|-------|---------|
| [Level 1 · Entry](level-1/index.md) | What Vue is, create-vue & Vite, template syntax, `ref`/`reactive`/`computed`, watchers & lifecycle, props & emits, forms & `v-model`, slots, styling & transitions | 9 topics + 1 project (a habit tracker) |
| [Level 2 · Intermediate](level-2/index.md) | Composables, provide/inject, Vue Router basics and in depth, Pinia, async data, form validation, TypeScript with Vue | 9 topics + 1 project (a routed recipe book) |
| [Level 3 · Advanced](level-3/index.md) | Reactivity internals, the rendering mechanism, directives & plugins, Teleport/KeepAlive/Suspense, code splitting, performance, testing, accessibility, error handling & security | 9 topics + 1 project (an accessible Kanban board) |
| [Level 4 · Master](level-4/index.md) | SSR from scratch, Nuxt, production builds, component libraries, large-app architecture, Vue 2 migration, i18n, deployment, observability | 9 topics + 1 capstone (a server-rendered meetup app) |

## How to use this site

- **Build as you read.** Scaffold a project in Level 1, lesson 02 and keep it open. Code
  blocks are titled with their file path (`src/stores/recipes.ts`) so you know where each
  piece lives, and each level's project reuses what the lessons built.
- **Open the browser devtools and the Vue devtools.** Many lessons ask you to watch what
  re-renders, what the network did, or what the compiler produced — seeing it beats
  reading about it.
- **Know modern JavaScript first.** Modules, destructuring, promises and `async`/`await`
  are assumed from lesson one; if they're shaky, start with
  [JavaScript Mastery Path](https://learnmasterypath.com/javascript-mastery-path/). Types
  appear from Level 2 onward — [TypeScript Mastery Path](https://learnmasterypath.com/typescript-mastery-path/)
  covers them. For the server side of SSR, see
  [Node.js Mastery Path](https://learnmasterypath.com/nodejs-mastery-path/); for
  end-to-end tests, [Playwright Mastery Path](https://learnmasterypath.com/playwright-mastery-path/);
  and to compare frameworks, [React Mastery Path](https://learnmasterypath.com/react-mastery-path/)
  and [Angular Mastery Path](https://learnmasterypath.com/angular-mastery-path/) build
  similar projects.
- Do the **Exercise** at the end of each lesson; the module-10 projects build on them.

Start here → [Level 1 · Entry](level-1/index.md)


🎥 **Prefer video?** Watch the [Mastery Path video series](https://youtube.com/@learnmasterypath) on YouTube — Shorts and full walkthroughs of lessons across the series. [▶ Subscribe free](https://www.youtube.com/@learnmasterypath?sub_confirmation=1)

## More from the Mastery Path series

Free, structured, module-wise training across 101 other languages, platforms and disciplines — expand a category to browse:

<div class="mastery-tracks">
<details class="mastery-card">
<summary>Languages <span class="mastery-count">20</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/python-mastery-path/">🐍 Python</a>
  <a href="https://learnmasterypath.com/java-mastery-path/">☕ Java</a>
  <a href="https://learnmasterypath.com/javascript-mastery-path/">🟨 JavaScript</a>
  <a href="https://learnmasterypath.com/typescript-mastery-path/">🔷 TypeScript</a>
  <a href="https://learnmasterypath.com/csharp-mastery-path/">🔵 C#</a>
  <a href="https://learnmasterypath.com/shell-mastery-path/">🐚 Shell/Bash</a>
  <a href="https://learnmasterypath.com/powershell-mastery-path/">💻 PowerShell</a>
  <a href="https://learnmasterypath.com/c-mastery-path/">🇨 C</a>
  <a href="https://learnmasterypath.com/cpp-mastery-path/">➕ C++</a>
  <a href="https://learnmasterypath.com/go-mastery-path/">🐹 Go</a>
  <a href="https://learnmasterypath.com/rust-mastery-path/">🦀 Rust</a>
  <a href="https://learnmasterypath.com/sql-mastery-path/">🗄️ SQL</a>
  <a href="https://learnmasterypath.com/ruby-mastery-path/">💎 Ruby</a>
  <a href="https://learnmasterypath.com/php-mastery-path/">🐘 PHP</a>
  <a href="https://learnmasterypath.com/kotlin-mastery-path/">🟣 Kotlin</a>
  <a href="https://learnmasterypath.com/swift-mastery-path/">🐦 Swift</a>
  <a href="https://learnmasterypath.com/dart-mastery-path/">🎯 Dart</a>
  <a href="https://learnmasterypath.com/scala-mastery-path/">🔴 Scala</a>
  <a href="https://learnmasterypath.com/r-mastery-path/">📊 R</a>
  <a href="https://learnmasterypath.com/matlab-mastery-path/">🟧 MATLAB</a>
</div>
</details>
<details class="mastery-card">
<summary>Web Frameworks <span class="mastery-count">10</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/react-mastery-path/">⚛️ React</a>
  <a href="https://learnmasterypath.com/nextjs-mastery-path/">▲ Next.js</a>
  <a href="https://learnmasterypath.com/angular-mastery-path/">🅰️ Angular</a>
  <a href="https://learnmasterypath.com/html-css-mastery-path/">🎨 HTML & CSS</a>
  <a href="https://learnmasterypath.com/tailwind-mastery-path/">🌬️ Tailwind CSS</a>
  <a href="https://learnmasterypath.com/django-mastery-path/">🎸 Django</a>
  <a href="https://learnmasterypath.com/fastapi-mastery-path/">🚀 FastAPI</a>
  <a href="https://learnmasterypath.com/graphql-mastery-path/">🕸️ GraphQL</a>
  <a href="https://learnmasterypath.com/nodejs-mastery-path/">🟩 Node.js</a>
  <a href="https://learnmasterypath.com/spring-boot-mastery-path/">🍃 Spring Boot</a>
</div>
</details>
<details class="mastery-card">
<summary>Mobile <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/flutter-mastery-path/">💙 Flutter</a>
  <a href="https://learnmasterypath.com/react-native-mastery-path/">📲 React Native</a>
</div>
</details>
<details class="mastery-card">
<summary>Testing & QA <span class="mastery-count">6</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/java-testing-mastery-path/">🧪 Java Testing</a>
  <a href="https://learnmasterypath.com/cpp-testing-mastery-path/">🧪 C/C++ Testing</a>
  <a href="https://learnmasterypath.com/python-testing-mastery-path/">🧪 Python Testing</a>
  <a href="https://learnmasterypath.com/automotive-testing-mastery-path/">🚗 Automotive Testing</a>
  <a href="https://learnmasterypath.com/selenium-mastery-path/">🧭 Selenium</a>
  <a href="https://learnmasterypath.com/cypress-mastery-path/">🌲 Cypress</a>
</div>
</details>
<details class="mastery-card">
<summary>Security <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/cybersecurity-mastery-path/">🛡️ Cybersecurity</a>
  <a href="https://learnmasterypath.com/ethical-hacking-mastery-path/">🎯 Ethical Hacking</a>
</div>
</details>
<details class="mastery-card">
<summary>Cloud Platforms <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/aws-mastery-path/">☁️ AWS</a>
  <a href="https://learnmasterypath.com/azure-mastery-path/">☁️ Azure</a>
  <a href="https://learnmasterypath.com/gcp-mastery-path/">☁️ GCP</a>
  <a href="https://learnmasterypath.com/ibm-cloud-mastery-path/">☁️ IBM Cloud</a>
  <a href="https://learnmasterypath.com/adobe-mastery-path/">🎨 Adobe</a>
</div>
</details>
<details class="mastery-card">
<summary>Data & Analytics <span class="mastery-count">7</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/data-engineering-mastery-path/">🛠️ Data Engineering</a>
  <a href="https://learnmasterypath.com/data-science-mastery-path/">📈 Data Science</a>
  <a href="https://learnmasterypath.com/tableau-mastery-path/">📊 Tableau</a>
  <a href="https://learnmasterypath.com/excel-mastery-path/">📗 Excel</a>
  <a href="https://learnmasterypath.com/pyspark-mastery-path/">⚡ PySpark</a>
  <a href="https://learnmasterypath.com/etl-datalake-mastery-path/">🏞️ ETL & Data Lake</a>
  <a href="https://learnmasterypath.com/power-bi-mastery-path/">📉 Power BI</a>
</div>
</details>
<details class="mastery-card">
<summary>Databases <span class="mastery-count">6</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/postgresql-mastery-path/">🐘 PostgreSQL</a>
  <a href="https://learnmasterypath.com/mongodb-mastery-path/">🍃 MongoDB</a>
  <a href="https://learnmasterypath.com/redis-mastery-path/">🟥 Redis</a>
  <a href="https://learnmasterypath.com/snowflake-mastery-path/">❄️ Snowflake</a>
  <a href="https://learnmasterypath.com/databricks-mastery-path/">🧱 Databricks</a>
  <a href="https://learnmasterypath.com/python-mastery-path/vector-databases-mastery-path/">🧭 Vector Databases</a>
</div>
</details>
<details class="mastery-card">
<summary>AI / ML / LLM <span class="mastery-count">12</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/ai-ml-mastery-path/">🤖 AI/ML</a>
  <a href="https://learnmasterypath.com/llm-dev-mastery-path/">🧠 LLM Dev</a>
  <a href="https://learnmasterypath.com/rag-mastery-path/">📚 RAG</a>
  <a href="https://learnmasterypath.com/edge-ai-mastery-path/">📱 Edge AI</a>
  <a href="https://learnmasterypath.com/claude-training-mastery-path/">🔶 Claude Training</a>
  <a href="https://learnmasterypath.com/ai-tools-mastery-path/">🧰 AI Tools</a>
  <a href="https://learnmasterypath.com/ml-math-mastery-path/">➗ ML Math Foundations</a>
  <a href="https://learnmasterypath.com/prompt-engineering-mastery-path/">✍️ Prompt Engineering</a>
  <a href="https://learnmasterypath.com/ai-agents-mastery-path/">🕹️ AI Agents</a>
  <a href="https://learnmasterypath.com/mlops-mastery-path/">🔁 MLOps</a>
  <a href="https://learnmasterypath.com/computer-vision-mastery-path/">👁️ Computer Vision</a>
  <a href="https://learnmasterypath.com/python-mastery-path/mcp-mastery-path/">🔌 MCP</a>
</div>
</details>
<details class="mastery-card">
<summary>Embedded Systems <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/embedded-mastery-path/">🔌 Embedded</a>
  <a href="https://learnmasterypath.com/embedded-linux-mastery-path/">🐧 Embedded Linux</a>
  <a href="https://learnmasterypath.com/embedded-python-mastery-path/">🐍 Embedded Python</a>
  <a href="https://learnmasterypath.com/freertos-mastery-path/">⏱️ FreeRTOS</a>
  <a href="https://learnmasterypath.com/s32k-mastery-path/">🔧 S32K</a>
</div>
</details>
<details class="mastery-card">
<summary>Leadership & Management <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/product-manager-mastery-path/">📋 Product Manager</a>
  <a href="https://learnmasterypath.com/product-lead-mastery-path/">🧭 Product Lead</a>
  <a href="https://learnmasterypath.com/project-manager-mastery-path/">📅 Project Manager</a>
  <a href="https://learnmasterypath.com/ai-manager-mastery-path/">🤖 AI Manager</a>
  <a href="https://learnmasterypath.com/servant-leadership-mastery-path/">🤝 Servant Leadership</a>
</div>
</details>
<details class="mastery-card">
<summary>Professional Skills <span class="mastery-count">4</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/english-fluency-mastery-path/">🗣️ English Fluency & IELTS</a>
  <a href="https://learnmasterypath.com/workday-mastery-path/">🧑‍💼 Workday</a>
  <a href="https://learnmasterypath.com/digital-marketing-mastery-path/">📣 Digital Marketing & SEO</a>
  <a href="https://learnmasterypath.com/tally-gst-mastery-path/">🧾 Tally Prime & GST</a>
</div>
</details>
<details class="mastery-card">
<summary>Careers & Interviews <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/dsa-interview-mastery-path/">🧩 DSA & Coding Interviews</a>
  <a href="https://learnmasterypath.com/system-design-mastery-path/">🏗️ System Design</a>
</div>
</details>
<details class="mastery-card">
<summary>Process & APIs <span class="mastery-count">3</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/agile-mastery-path/">🔄 Agile/Scrum/Kanban</a>
  <a href="https://learnmasterypath.com/rest-api-mastery-path/">🔗 REST API</a>
  <a href="https://learnmasterypath.com/playwright-mastery-path/">🎭 Playwright</a>
</div>
</details>
<details class="mastery-card">
<summary>Infrastructure & Ops <span class="mastery-count">12</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/server-ops-mastery-path/">🖥️ Server Ops</a>
  <a href="https://learnmasterypath.com/linux-mastery-path/">🐧 Linux</a>
  <a href="https://learnmasterypath.com/nodemcu-mastery-path/">📶 NodeMCU/IoT</a>
  <a href="https://learnmasterypath.com/terraform-mastery-path/">🌍 Terraform</a>
  <a href="https://learnmasterypath.com/kubernetes-mastery-path/">☸️ Kubernetes</a>
  <a href="https://learnmasterypath.com/docker-mastery-path/">🐳 Docker</a>
  <a href="https://learnmasterypath.com/github-mastery-path/">🐙 GitHub & Git</a>
  <a href="https://learnmasterypath.com/cicd-mastery-path/">🔁 CI/CD</a>
  <a href="https://learnmasterypath.com/ansible-mastery-path/">🤖 Ansible</a>
  <a href="https://learnmasterypath.com/kafka-mastery-path/">📨 Kafka</a>
  <a href="https://learnmasterypath.com/microservices-mastery-path/">🧩 Microservices</a>
  <a href="https://learnmasterypath.com/observability-mastery-path/">📈 Observability</a>
</div>
</details>
</div>
