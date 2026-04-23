<template>
  <div id="app">
    <h1>PDF Form Filler</h1>

    <div v-if="!pdfLoaded" class="upload-section">
      <p>Select a PDF file with form fields to fill:</p>
      <input type="file" accept="application/pdf" @change="onFileSelected" />
    </div>

    <div v-if="pdfLoaded" class="pdf-section">
      <h2>Fill the form fields below</h2>

      <div class="form-fields">
        <div v-for="field in formFields" :key="field.name" class="field-row">
          <label :for="field.name">{{ field.name }}</label>

          <input
            v-if="field.type === 'text'"
            :id="field.name"
            type="text"
            v-model="field.value"
          />

          <select
            v-if="field.type === 'dropdown'"
            :id="field.name"
            v-model="field.value"
          >
            <option v-for="opt in field.options" :key="opt" :value="opt">
              {{ opt }}
            </option>
          </select>

          <label v-if="field.type === 'checkbox'" class="checkbox-label">
            <input type="checkbox" v-model="field.value" />
          </label>

          <div v-if="field.type === 'radio'" class="radio-group">
            <label v-for="opt in field.options" :key="opt">
              <input
                type="radio"
                :name="field.name"
                :value="opt"
                v-model="field.value"
              />
              {{ opt }}
            </label>
          </div>
        </div>
      </div>

      <div v-if="formFields.length === 0" class="no-fields">
        <p>No fillable form fields found in this PDF.</p>
      </div>

      <div class="pdf-preview">
        <h3>PDF Preview</h3>
        <canvas ref="pdfCanvas"></canvas>
      </div>

      <div class="actions">
        <button @click="saveAndUpload" :disabled="saving">
          {{ saving ? 'Saving...' : 'Save & Upload Filled PDF' }}
        </button>
        <button @click="reset" class="secondary">Upload Another PDF</button>
      </div>
    </div>

    <div v-if="message" :class="['message', messageType]">
      {{ message }}
    </div>
  </div>
</template>

<script>
import { PDFDocument, PDFTextField, PDFCheckBox, PDFDropdown, PDFRadioGroup, PDFButton, PDFOptionList, PDFSignature } from 'pdf-lib';
import * as pdfjsLib from 'pdfjs-dist';
import axios from 'axios';

pdfjsLib.GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();

export default {
  name: 'App',
  data() {
    return {
      originalFilename: '',
      pdfBytes: null,
      pdfLoaded: false,
      formFields: [],
      saving: false,
      message: '',
      messageType: 'success',
    };
  },
  methods: {
    async onFileSelected(event) {
      const file = event.target.files[0];
      if (!file) return;

      this.message = '';

      try {
        const arrayBuffer = await file.arrayBuffer();
        this.pdfBytes = new Uint8Array(arrayBuffer);
        this.originalFilename = file.name;

        await this.extractFormFields();

        this.pdfLoaded = true;
        await this.$nextTick();
        await this.renderPreview();
        this.showMessage('PDF loaded. Fill the form fields and click "Save & Upload".', 'success');
      } catch (err) {
        this.showMessage('Failed to load PDF: ' + err.message, 'error');
      }
    },

    async extractFormFields() {
      const pdfDoc = await PDFDocument.load(this.pdfBytes);
      const form = pdfDoc.getForm();
      const fields = form.getFields();

      this.formFields = fields
        .filter((field) => !(field instanceof PDFButton) && !(field instanceof PDFSignature))
        .map((field) => {
          const name = field.getName();

          if (field instanceof PDFTextField) {
            return { name, type: 'text', value: field.getText() || '' };
          }
          if (field instanceof PDFCheckBox) {
            return { name, type: 'checkbox', value: field.isChecked() };
          }
          if (field instanceof PDFDropdown) {
            return {
              name,
              type: 'dropdown',
              value: field.getSelected()?.[0] || '',
              options: field.getOptions(),
            };
          }
          if (field instanceof PDFOptionList) {
            return {
              name,
              type: 'dropdown',
              value: field.getSelected()?.[0] || '',
              options: field.getOptions(),
            };
          }
          if (field instanceof PDFRadioGroup) {
            return {
              name,
              type: 'radio',
              value: field.getSelected() || '',
              options: field.getOptions(),
            };
          }
          // Fallback for other field types
          return { name, type: 'text', value: '' };
        });
    },

    async renderPreview() {
      const loadingTask = pdfjsLib.getDocument({ data: this.pdfBytes.slice() });
      const pdf = await loadingTask.promise;
      const page = await pdf.getPage(1);

      const scale = 1.5;
      const viewport = page.getViewport({ scale });
      const canvas = this.$refs.pdfCanvas;
      const context = canvas.getContext('2d');
      canvas.height = viewport.height;
      canvas.width = viewport.width;

      await page.render({ canvasContext: context, viewport }).promise;
    },

    async saveAndUpload() {
      this.saving = true;
      this.message = '';

      try {
        const pdfDoc = await PDFDocument.load(this.pdfBytes);
        const form = pdfDoc.getForm();

        for (const field of this.formFields) {
          try {
            if (field.type === 'text') {
              const f = form.getTextField(field.name);
              f.setText(field.value);
            } else if (field.type === 'checkbox') {
              const f = form.getCheckBox(field.name);
              if (field.value) f.check();
              else f.uncheck();
            } else if (field.type === 'dropdown') {
              const f = form.getDropdown(field.name);
              if (field.value) f.select(field.value);
            } else if (field.type === 'radio') {
              const f = form.getRadioGroup(field.name);
              if (field.value) f.select(field.value);
            }
          } catch (e) {
            console.warn(`Could not set field "${field.name}":`, e);
          }
        }

        const filledPdfBytes = await pdfDoc.save();
        const base64 = btoa(
          filledPdfBytes.reduce((data, byte) => data + String.fromCharCode(byte), '')
        );

        await axios.post('/api/save', {
          pdfData: `data:application/pdf;base64,${base64}`,
          originalFilename: this.originalFilename,
        });

        this.showMessage('Filled PDF saved and uploaded successfully!', 'success');
      } catch (err) {
        this.showMessage('Failed to save PDF: ' + err.message, 'error');
      } finally {
        this.saving = false;
      }
    },

    reset() {
      this.originalFilename = '';
      this.pdfBytes = null;
      this.pdfLoaded = false;
      this.formFields = [];
      this.message = '';
    },

    showMessage(text, type) {
      this.message = text;
      this.messageType = type;
    },
  },
};
</script>

<style>
#app {
  font-family: Arial, sans-serif;
  max-width: 900px;
  margin: 0 auto;
  padding: 20px;
}

h1 {
  color: #2c3e50;
  border-bottom: 2px solid #42b983;
  padding-bottom: 10px;
}

.upload-section {
  margin: 30px 0;
  padding: 30px;
  border: 2px dashed #ccc;
  border-radius: 8px;
  text-align: center;
}

.upload-section input[type='file'] {
  margin: 10px 0;
}

button {
  background-color: #42b983;
  color: white;
  border: none;
  padding: 10px 24px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  margin: 5px;
}

button:hover {
  background-color: #38a373;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

button.secondary {
  background-color: #6c757d;
}

button.secondary:hover {
  background-color: #5a6268;
}

.form-fields {
  margin: 20px 0;
}

.field-row {
  display: flex;
  align-items: center;
  margin: 10px 0;
  padding: 8px;
  background: #f8f9fa;
  border-radius: 4px;
}

.field-row label {
  min-width: 200px;
  font-weight: bold;
  margin-right: 10px;
}

.field-row input[type='text'],
.field-row select {
  flex: 1;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 14px;
}

.checkbox-label {
  min-width: auto !important;
}

.radio-group label {
  min-width: auto;
  font-weight: normal;
  margin-right: 15px;
}

.no-fields {
  padding: 20px;
  background: #fff3cd;
  border-radius: 4px;
  text-align: center;
}

.pdf-preview {
  margin: 20px 0;
  text-align: center;
}

.pdf-preview canvas {
  border: 1px solid #ccc;
  max-width: 100%;
  height: auto;
}

.actions {
  margin: 20px 0;
  text-align: center;
}

.message {
  margin: 20px 0;
  padding: 12px;
  border-radius: 4px;
  text-align: center;
}

.message.success {
  background-color: #d4edda;
  color: #155724;
}

.message.error {
  background-color: #f8d7da;
  color: #721c24;
}
</style>
