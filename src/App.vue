<template>
  <div class="container">
    <div class="card border-0">
      <div class="card-body">
        <h1 class="text-center card-title mb-4">Word fayl tahlilchisi</h1>

        <!-- File Upload Input -->
        <div class="mb-3">
          <label for="fileInput" class="form-label h5">Word faylini yuklang</label>
          <input type="file" id="fileInput" @change="handleFile" class="form-control" accept=".docx" />
        </div>

        <!-- Sorting Buttons -->
        <div v-if="wordFrequency.length" class="row mx-1">
          <button @click="sortBy('asc')" class="btn btn-outline-primary col mx-1">
            Soni bo'yicha saralash (o'sish bo'yicha)
          </button>
          <button @click="sortBy('desc')" class="btn btn-outline-secondary col">
            Soni bo'yicha saralash (kamayish bo'yicha)
          </button>
          <button @click="sortBy('alphabet')" class="btn btn-outline-success col mx-1">
            Alifbo tartibida tartiblash
          </button>

          <button @click="downloadCSV" class="btn btn-primary col">
            Jadvalni CSV sifatida yuklab oling
          </button>

          <button @click="downloadDocx" class="btn btn-primary col mx-1">
            Word jadvalini yuklab oling
          </button>
        </div>

        <div v-if="wordFrequency.length" class="card card-body my-2">
          {{  word }}
        </div>

        <!-- Word Frequency Table -->
        <div v-if="wordFrequency.length" class="table-responsive mt-4">
          <table class="table table-bordered table-striped text-center">
            <thead class="table-dark">
              <tr>
                <th>So'z</th>
                <th>Takrorlanishlar soni</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(item, index) in paginatedData" :key="index">
                <td>{{ item.word }}</td>
                <td>{{ item.count }}</td>
              </tr>
            </tbody>
          </table>
        </div>
        <!-- Pagination Controls -->
        <div v-if="wordFrequency.length" class="d-flex justify-content-center gap-2 mt-3">
          <button class="btn btn-outline-primary" :disabled="currentPage === 1" @click="changePage(currentPage - 1)">
            Oldingi
          </button>
          <span class="align-self-center">
            Sahifa: {{ currentPage }} / {{ totalPages }}
          </span>
          <button class="btn btn-outline-primary" :disabled="currentPage === totalPages"
            @click="changePage(currentPage + 1)">
            Keyingisi
          </button>
        </div>
      </div>
    </div>


  </div>
</template>

<script>
  import mammoth from "mammoth";
  import {
    saveAs
  } from "file-saver";
  export default {
    data() {
      return {
        wordFrequency: [],
        currentPage: 1,
        itemsPerPage: 10,
        word: ""
      };
    },

    computed: {
      totalPages() {
        return Math.ceil(this.wordFrequency.length / this.itemsPerPage);
      },

      paginatedData() {
        const startIndex = (this.currentPage - 1) * this.itemsPerPage;
        const endIndex = startIndex + this.itemsPerPage;
        return this.wordFrequency.slice(startIndex, endIndex);
      },
    },
    methods: {
      async handleFile(event) {
        const file = event.target.files[0];
        if (file) {
          const reader = new FileReader();
          reader.onload = async (e) => {
            const arrayBuffer = e.target.result;
            const result = await mammoth.extractRawText({
              arrayBuffer
            });
            this.word = result.value
            this.processText(result.value);
          };
          reader.readAsArrayBuffer(file);
        }
      },

      sortBy(order) {
        if (order === "asc") {
          this.wordFrequency.sort((a, b) => a.count - b.count);
        } else if (order === "desc") {
          this.wordFrequency.sort((a, b) => b.count - a.count);
        } else if (order === "alphabet") {
          this.wordFrequency.sort((a, b) => a.word.localeCompare(b.word));
        }
      },

      processText(text) {
        const words = text
          .toLowerCase()
          .replace(/[^a-zA-Z0-9\s]/g, "")
          .split(/\s+/)
          .filter(Boolean);
        const frequencyMap = words.reduce((map, word) => {
          map[word] = (map[word] || 0) + 1;
          return map;
        }, {});
        this.wordFrequency = Object.entries(frequencyMap)
          .map(([word, count]) => ({
            word,
            count
          }))
          .sort((a, b) => b.count - a.count);
      },

      changePage(page) {
        if (page < 1 || page > this.totalPages) return;
        this.currentPage = page;
      },

      downloadCSV() {
        const csvContent =
          "data:text/csv;charset=utf-8," + ["Word,Frequency"]
          .concat(
            this.wordFrequency.map((item) => `${item.word},${item.count}`)
          )
          .join("\n");
        const blob = new Blob([decodeURIComponent(encodeURI(csvContent))], {
          type: "text/csv;charset=utf-8;",
        });
        saveAs(blob, "word-frequency.csv");
      },

      
      downloadDocx() {

      }
    },
  };
</script>