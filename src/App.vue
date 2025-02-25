<template>
  <div class="container">
    <div class="card my-2">
      <div class="card-body">
        <h1 class="card-title">So'z tahlilchisi</h1>

        <!-- File Upload Input -->
        <div class="mb-3">
          <label for="fileInput" class="form-label">Word faylini yuklang</label>
          <input type="file" id="fileInput" @change="handleFile" class="form-control" accept=".docx" />
        </div>

        <div v-if="wordFrequency.length" class="card card-body my-2">
          {{  word }}
        </div>

        <!-- Sorting Buttons -->
        <div v-if="wordFrequency.length" class="row g-2">
          <div class="col-12 col-md-6 col-lg-4">
            <button @click="sortBy('asc')" class="btn btn-outline-primary w-100">
              Soni bo'yicha saralash (o'sish bo'yicha)
            </button>
          </div>
          <div class="col-12 col-md-6 col-lg-4">
            <button @click="sortBy('desc')" class="btn btn-outline-secondary w-100">
              Soni bo'yicha saralash (kamayish bo'yicha)
            </button>
          </div>
          <div class="col-12 col-md-6 col-lg-4">
            <button @click="sortBy('alphabet')" class="btn btn-outline-success w-100">
              Alifbo tartibida tartiblash
            </button>
          </div>
          <div class="col-12 col-md-6 col-lg-4">
            <button @click="downloadCSV" class="btn btn-primary w-100">
              Jadvalni CSV sifatida yuklab oling
            </button>
          </div>
          <div class="col-12 col-md-6 col-lg-4">
            <button @click="downloadDocx" class="btn btn-primary w-100">
              Word jadvalini yuklab oling
            </button>
          </div>
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
    Document,
    Packer,
    Paragraph,
    Table,
    TableRow,
    TableCell,
    WidthType
  } from "docx";
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
        const doc = new Document({
          sections: [{
            properties: {},
            children: [
              new Paragraph("So'z Taxlil jadvali"),
              new Table({
                width: {
                  size: 100,
                  type: WidthType.PERCENTAGE
                },
                rows: [
                  new TableRow({
                    children: [
                      new TableCell({
                        children: [new Paragraph("So'z")],
                      }),
                      new TableCell({
                        children: [new Paragraph("Takrorlanishlar soni")],
                      }),
                    ],
                  }),
                  ...this.wordFrequency.map(
                    (item) =>
                    new TableRow({
                      children: [
                        new TableCell({
                          children: [new Paragraph(item.word)],
                        }),
                        new TableCell({
                          children: [new Paragraph(item.count.toString())],
                        }),
                      ],
                    })
                  ),
                ],
              }),
            ],
          }, ],
        });

        Packer.toBlob(doc).then((blob) => {
          saveAs(blob, "word-frequency.docx");
        });
      }
    },
  };
</script>