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
        <template v-for="item in displayFields" :key="item.key">
          <!-- Checkbox group -->
          <div v-if="item.type === 'checkbox-group'" class="field-row checkbox-group-row">
            <label>{{ item.title }}</label>
            <div class="checkbox-group">
              <label v-for="(cb, idx) in item.checkboxes" :key="cb.name" class="checkbox-label">
                <input type="checkbox" v-model="cb.value" />
                {{ cb.label || ('Checkbox ' + (idx + 1)) }}
              </label>
            </div>
          </div>

          <!-- Single checkbox -->
          <div v-else-if="item.type === 'checkbox'" class="field-row">
            <label :for="item.name">{{ item.name }}</label>
            <label class="checkbox-label">
              <input type="checkbox" v-model="item.value" />
              {{ item.label || item.name }}
            </label>
          </div>

          <!-- Other fields -->
          <div v-else class="field-row">
            <label :for="item.name">{{ item.name }}</label>

            <input
              v-if="item.type === 'text'"
              :id="item.name"
              type="text"
              v-model="item.value"
            />

            <textarea
              v-if="item.type === 'textarea'"
              :id="item.name"
              v-model="item.value"
              rows="4"
            ></textarea>

            <select
              v-if="item.type === 'dropdown'"
              :id="item.name"
              v-model="item.value"
            >
              <option v-for="opt in item.options" :key="opt" :value="opt">
                {{ opt }}
              </option>
            </select>

            <div v-if="item.type === 'radio'" class="radio-group">
              <label v-for="opt in item.options" :key="opt">
                <input
                  type="radio"
                  :name="item.name"
                  :value="opt"
                  v-model="item.value"
                />
                {{ opt }}
              </label>
            </div>
          </div>
        </template>
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
  computed: {
    displayFields() {
      const result = [];
      let i = 0;
      const fields = this.formFields;
      while (i < fields.length) {
        if (fields[i].type === 'checkbox') {
          // Collect consecutive checkboxes
          const group = [];
          const startIdx = i;
          while (i < fields.length && fields[i].type === 'checkbox') {
            group.push(fields[i]);
            i++;
          }
          if (group.length > 1) {
            // Find common prefix for group title
            const names = group.map(f => f.name);
            let commonPrefix = names[0];
            for (let n = 1; n < names.length; n++) {
              while (!names[n].startsWith(commonPrefix)) {
                commonPrefix = commonPrefix.slice(0, -1);
              }
            }
            // Clean up prefix (remove trailing dots, spaces, numbers)
            let title = commonPrefix.replace(/[\s._-]+$/, '');
            if (!title) title = group[0].name;
            result.push({
              key: 'cbgroup-' + startIdx,
              type: 'checkbox-group',
              title,
              checkboxes: group,
            });
          } else {
            result.push({ ...group[0], key: group[0].name });
          }
        } else {
          result.push({ ...fields[i], key: fields[i].name });
          i++;
        }
      }
      return result;
    },
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
            const isMultiline = field.isMultiline();
            return { name, type: isMultiline ? 'textarea' : 'text', value: field.getText() || '' };
          }
          if (field instanceof PDFCheckBox) {
            const widgets = field.acroField.getWidgets();
            let label = name;
            if (widgets.length > 0) {
              const onValue = widgets[0].getOnValue()?.decodeText?.() || widgets[0].getOnValue()?.toString?.() || '';
              if (onValue && onValue !== 'Yes') label = onValue;
            }
            return { name, type: 'checkbox', value: field.isChecked(), label };
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
            if (field.type === 'text' || field.type === 'textarea') {
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
.field-row textarea,
.field-row select {
  flex: 1;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 14px;
  box-sizing: border-box;
  min-width: 0;
  width: 0;
}

.field-row textarea {
  font-family: Arial, sans-serif;
  resize: vertical;
}

.checkbox-label {
  min-width: auto !important;
}

.checkbox-group {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.checkbox-group-row {
  align-items: flex-start;
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
