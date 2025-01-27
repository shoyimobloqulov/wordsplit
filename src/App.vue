<template>
  <div class="container py-4">
    <div class="card border-0">
      <div class="card-body">
        <h1 class="text-center card-title mb-4">Word fayl tahlilchisi</h1>

        <!-- File Upload Input -->
        <div class="mb-3">
          <label for="fileInput" class="form-label h5">Word faylini yuklang</label>
          <input
            type="file"
            id="fileInput"
            @change="handleFile"
            class="form-control"
            accept=".docx"
          />
        </div>

        <!-- Sorting Buttons -->
        <div v-if="wordFrequency.length" class="row">
          <button @click="sortBy('asc')" class="btn btn-outline-primary col mx-1">
            Sort by Frequency (Ascending)
          </button>
          <button @click="sortBy('desc')" class="btn btn-outline-secondary col">
            Sort by Frequency (Descending)
          </button>
          <button @click="sortBy('alphabet')" class="btn btn-outline-success col mx-1">
            Sort Alphabetically
          </button>

          <button @click="downloadCSV" class="btn btn-primary col">
              Download Table as CSV
          </button>

          <button @click="downloadDocx" class="btn btn-primary col mx-1">
            Download Table Word
          </button>
        </div>

        <!-- Word Frequency Table -->
        <div v-if="wordFrequency.length" class="table-responsive mt-4">
          <table class="table table-bordered table-striped text-center">
            <thead class="table-dark">
              <tr>
                <th>Word</th>
                <th>Frequency</th>
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
        <button 
          class="btn btn-outline-primary" 
          :disabled="currentPage === 1" 
          @click="changePage(currentPage - 1)">
          Oldingi
        </button>
        <span class="align-self-center">
          Sahifa: {{ currentPage }} - {{ totalPages }}
        </span>
        <button 
          class="btn btn-outline-primary" 
          :disabled="currentPage === totalPages" 
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
import { Document, Packer, Table, TableRow, TableCell, TextRun } from "docx";
import { saveAs } from "file-saver";
import Docxtemplater from "docxtemplater";
import PizZip from "pizzip";
export default {
  data() {
    return {
      wordFrequency: [],
      currentPage: 1, 
      itemsPerPage: 10, 
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
          const result = await mammoth.extractRawText({ arrayBuffer });
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
        .map(([word, count]) => ({ word, count }))
        .sort((a, b) => b.count - a.count);
    },

    changePage(page) {
      if (page < 1 || page > this.totalPages) return;
      this.currentPage = page;
    },

    downloadCSV() {
      const csvContent =
        "data:text/csv;charset=utf-8," +
        ["Word,Frequency"]
          .concat(
            this.wordFrequency.map((item) => `${item.word},${item.count}`)
          )
          .join("\n");
      const blob = new Blob([decodeURIComponent(encodeURI(csvContent))], {
        type: "text/csv;charset=utf-8;",
      });
      saveAs(blob, "word-frequency.csv");
    },
    async downloadDocx() {
      if (this.wordFrequency.length === 0) {
        console.error("No word frequency data available!");
        alert("No word frequency data available!");
        return;
      }

      // Ensure wordFrequency contains valid data
      console.log("Word Frequency Data: ", this.wordFrequency);

      // Create the document
      const doc = new Document({
        sections: [
          {
            properties: {},
            children: [
              new Table({
                rows: [
                  new TableRow({
                    children: [
                      new TableCell({
                        children: [new TextRun("Word")],
                      }),
                      new TableCell({
                        children: [new TextRun("Frequency")],
                      }),
                    ],
                  }),
                  ...this.wordFrequency.map((item) =>
                    new TableRow({
                      children: [
                        new TableCell({
                          children: [new TextRun(item.word)],
                        }),
                        new TableCell({
                          children: [new TextRun(item.count.toString())],
                        }),
                      ],
                    })
                  ),
                ],
              }),
            ],
          },
        ],
      });

      try {
        // Check if the document is being created correctly
        const blob = await Packer.toBlob(doc);
        console.log("DOCX file generated successfully!");

        // Log the Blob content for debugging (optional)
        const reader = new FileReader();
        reader.onload = function (event) {
          console.log("Blob Content: ", event.target.result);
        };
        reader.readAsText(blob);

        // Save the DOCX file
        saveAs(blob, "word-frequency.docx");
      } catch (error) {
        // Handle any errors during document generation
        console.error("Error generating DOCX:", error);
        alert("An error occurred while generating the Word document. Please try again.");
      }
    }

  },
};
</script>

