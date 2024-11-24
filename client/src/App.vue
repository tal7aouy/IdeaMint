<script setup lang="ts">
import { ChatAnthropic } from "@langchain/anthropic"
import { ChatPromptTemplate } from "@langchain/core/prompts"
import { StringOutputParser } from "@langchain/core/output_parsers"
import html2pdf from "html2pdf.js"
import { ref, reactive, onMounted } from "vue"
const isDarkMode = ref(false)

onMounted(() => {
  const savedTheme = localStorage.getItem("theme")
  isDarkMode.value =
    savedTheme === "dark" ||
    (!savedTheme && window.matchMedia("(prefers-color-scheme: dark)").matches)
  applyTheme()
})

const toggleTheme = () => {
  isDarkMode.value = !isDarkMode.value
  applyTheme()
  localStorage.setItem("theme", isDarkMode.value ? "dark" : "light")
}

const applyTheme = () => {
  document.documentElement.classList.toggle("dark", isDarkMode.value)
}
const form = reactive({
  title: "",
  tags: "",
  targetWordCount: 800,
  tone: "professional",
  format: "blog",
})
const isGenerating = ref(false)
const generatedDescription = ref("")

const generateDescription = async () => {
  isGenerating.value = true

  try {
    const model = new ChatAnthropic({
      apiKey: import.meta.env.VITE_ANTHROPIC_API_KEY,
      modelName: "claude-3-5-sonnet-20240620",
    })
    const parser = new StringOutputParser()

    const prompt = ChatPromptTemplate.fromMessages([
      [
        "human",
        `Generate a detailed content idea description for a {format}.
        Title: {title}
        Tags: {tags}
        Target Word Count: {targetWordCount} words
        Tone: {tone}

        Please provide a compelling description that includes:
        - Main topic overview
        - Key points to cover
        - Target audience
        - Potential value for readers
        - Suggested outline with section headers
        - SEO recommendations
        - Potential sources/references to cite

        Description:`,
      ],
    ])

    const formattedPrompt = await prompt.format({
      title: form.title,
      tags: form.tags,
      targetWordCount: form.targetWordCount,
      tone: form.tone,
      format: form.format,
    })

    const response = await model.invoke(formattedPrompt)
    generatedDescription.value = await parser.invoke(response)
  } catch (error) {
    console.error("Error generating description:", error)
  } finally {
    isGenerating.value = false
  }
}

const downloadAsPDF = () => {
  const tempDiv = document.createElement("div")
  tempDiv.innerHTML = `
    <h1>${form.title}</h1>
    <div style="white-space: pre-wrap;">${generatedDescription.value}</div>
  `

  const options = {
    margin: 1,
    filename: `${form.title || "generated-description"}.pdf`,
    html2canvas: { scale: 2 },
    jsPDF: { unit: "in", format: "letter", orientation: "portrait" },
  }

  html2pdf().set(options).from(tempDiv).save()
}

const downloadAsHTML = () => {
  const html = `
    <html>
      <head>
        <title>${form.title || "Generated Description"}</title>
        <style>
          body { font-family: Arial, sans-serif; padding: 20px; }
          .description { white-space: pre-wrap; }
        </style>
      </head>
      <body>
        <h1>${form.title}</h1>
        <div class="description">${generatedDescription.value}</div>
      </body>
    </html>
  `

  const blob = new Blob([html], { type: "text/html" })
  const url = window.URL.createObjectURL(blob)
  const a = document.createElement("a")
  a.href = url
  a.download = `${form.title || "generated-description"}.html`
  a.click()
  window.URL.revokeObjectURL(url)
}
</script>

<template>
  <div clas="min-h-screen bg-gray-100 dark:bg-gray-900">
    <div class="fixed top-4 right-4">
      <button
        @click="toggleTheme"
        class="p-2 rounded-lg bg-gray-200 dark:bg-gray-700 hover:bg-gray-300 dark:hover:bg-gray-600"
      >
        <svg
          v-if="isDarkMode"
          class="w-6 h-6 text-yellow-500"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z"
          />
        </svg>
        <svg
          v-else
          class="w-6 h-6 text-gray-700"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M20.354 15.354A9 9 0 018.646 3.646 9.003 9.003 0 0012 21a9.003 9.003 0 008.354-5.646z"
          />
        </svg>
      </button>
    </div>
    <div class="py-12">
      <div class="mx-auto max-w-7xl sm:px-6 lg:px-8">
        <div
          class="overflow-hidden bg-white dark:bg-gray-800 shadow-sm sm:rounded-lg"
        >
          <div class="p-6 text-gray-900 dark:text-gray-100">
            <form @submit.prevent="generateDescription" class="space-y-4">
              <div>
                <label
                  for="title"
                  class="block text-sm font-medium text-gray-700 dark:text-gray-300"
                  >Title</label
                >
                <input
                  id="title"
                  v-model="form.title"
                  type="text"
                  placeholder="Enter your title here"
                  class="w-full pl-3 pr-3 py-2 text-gray-500 dark:text-gray-300 bg-transparent outline-none border dark:border-gray-600 focus:border-indigo-600 dark:focus:border-indigo-500 shadow-sm rounded-lg dark:bg-gray-700"
                  required
                />
              </div>

              <div>
                <label
                  for="tags"
                  class="block text-sm font-medium text-gray-700 dark:text-gray-300"
                  >Tags (comma-separated)</label
                >
                <input
                  id="tags"
                  v-model="form.tags"
                  type="text"
                  class="w-full pl-3 pr-3 py-2 text-gray-500 dark:text-gray-300 bg-transparent outline-none border dark:border-gray-600 focus:border-indigo-600 dark:focus:border-indigo-500 shadow-sm rounded-lg dark:bg-gray-700"
                  placeholder="tech, programming, web development"
                />
              </div>
              <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
                <div>
                  <label
                    for="format"
                    class="block text-sm font-medium text-gray-700 dark:text-gray-300"
                    >Format</label
                  >
                  <select
                    id="format"
                    v-model="form.format"
                    class="w-full pl-3 pr-3 py-2 text-gray-500 dark:text-gray-300 bg-transparent outline-none border dark:border-gray-600 focus:border-indigo-600 dark:focus:border-indigo-500 shadow-sm rounded-lg dark:bg-gray-700"
                  >
                    <option value="blog">Blog Post</option>
                    <option value="article">Article</option>
                    <option value="whitepaper">White Paper</option>
                    <option value="case-study">Case Study</option>
                  </select>
                </div>

                <div>
                  <label
                    for="tone"
                    class="block text-sm font-medium text-gray-700 dark:text-gray-300"
                    >Tone</label
                  >
                  <select
                    id="tone"
                    v-model="form.tone"
                    class="w-full pl-3 pr-3 py-2 text-gray-500 dark:text-gray-300 bg-transparent outline-none border dark:border-gray-600 focus:border-indigo-600 dark:focus:border-indigo-500 shadow-sm rounded-lg dark:bg-gray-700"
                  >
                    <option value="professional">Professional</option>
                    <option value="casual">Casual</option>
                    <option value="academic">Academic</option>
                    <option value="conversational">Conversational</option>
                  </select>
                </div>

                <div>
                  <label
                    for="wordCount"
                    class="block text-sm font-medium text-gray-700 dark:text-gray-300"
                    >Target Word Count</label
                  >
                  <input
                    id="wordCount"
                    v-model="form.targetWordCount"
                    type="number"
                    min="100"
                    step="100"
                    class="w-full pl-3 pr-3 py-2 text-gray-500 dark:text-gray-300 bg-transparent outline-none border dark:border-gray-600 focus:border-indigo-600 dark:focus:border-indigo-500 shadow-sm rounded-lg dark:bg-gray-700"
                  />
                </div>
              </div>
              <button
                type="submit"
                :disabled="isGenerating"
                class="inline-flex justify-center rounded-md border border-transparent bg-indigo-600 px-4 py-2 text-sm font-medium text-white shadow-sm hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2"
              >
                {{ isGenerating ? "Generating..." : "Generate Description" }}
              </button>
            </form>

            <div v-if="generatedDescription" class="mt-6">
              <div class="flex items-center justify-between">
                <div class="space-x-2">
                  <button
                    @click="downloadAsPDF"
                    class="inline-flex items-center rounded-md bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700"
                  >
                    Download PDF
                  </button>
                  <button
                    @click="downloadAsHTML"
                    class="inline-flex items-center rounded-md bg-green-600 px-4 py-2 text-sm font-medium text-white hover:bg-green-700"
                  >
                    Download HTML
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
