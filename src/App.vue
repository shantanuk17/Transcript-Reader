<template>
  <div class="container">
    <h1>Read Data From Transcript</h1>
    <input type="file" accept="image/*,.pdf" @change="handleFileChange" />
    <button @click="sendToOpenAI" :disabled="!fileData || loading">
      {{ loading ? 'Processing...' : 'Extract Transcript Data' }}
    </button>
    <div v-if="error" class="error">{{ error }}</div>
    <div v-if="resultJSON">
      <table>
        <thead>
          <tr>
            <th>Course Code</th>
            <th>Name</th>
            <th>Credits</th>
            <th>Grade</th>
            <th>Semester</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(course, index) in resultJSON.courses" :key="index">
            <td>{{ course.code }}</td>
            <td>{{ course.name }}</td>
            <td>{{ course.credits }}</td>
            <td>{{ course.grade }}</td>
            <td>{{ course.semester }}</td>
          </tr>
        </tbody>
      </table>
      <div v-if="resultJSON.GPA" class="gpa">
        <strong>Overall GPA:</strong> {{ resultJSON.GPA }}
      </div>
    </div>
  </div>
</template>


<script setup>
import { ref } from 'vue';

const fileData = ref(null);
const loading = ref(false);
const error = ref('');
const resultJSON = ref(null);
const fileName = ref('');
const fileType = ref(''); 

const apiKey = import.meta.env.VITE_OPENAI_API_KEY;

function handleFileChange(event) {
  const file = event.target.files[0];
  if (!file) return;
  
  fileName.value = file.name;
  fileType.value = file.type; 
  
  const reader = new FileReader();
  reader.onload = () => {
    fileData.value = reader.result;
  };
  reader.readAsDataURL(file);
  // this.sendToOpenAI();
}

async function sendToOpenAI() {
  if (!fileData.value) return;
  loading.value = true;
  error.value = '';
  resultJSON.value = null;

  try {
    let content;
    if (fileType.value.startsWith('image/')) {
      content = [
        { type: 'text', text: 'Extract academic transcript data from this image' },
        { 
          type: 'image_url', 
          image_url: { 
            url: fileData.value,
            detail: 'high' 
          }
        }
      ];
    } else if (fileType.value === 'application/pdf') {
      const pdfText = await extractTextFromPDF(fileData.value);
      console.log('pdfText--', pdfText);
      content = [
        { type: 'text', text: `Extract academic transcript data from this PDF:\n\n${pdfText}` }
      ];
    } else {
      throw new Error('Unsupported file type. Please upload an image or PDF.');
    }

    const payload = {
      model: 'gpt-4o',
      messages: [
        { 
          role: 'system', 
          content: 'You are an academic transcript specialist. Extract structured data and return ONLY valid JSON with these keys: courses (array of objects with code, name, credits, grade, semester), and GPA (number).' 
        },
        { role: 'user', content: content }
      ],
      response_format: { type: "json_object" },
      max_tokens: 2000,
      temperature: 0.1
    };

    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey}`
      },
      body: JSON.stringify(payload)
    });

    const result = await response.json();
    
    if (result.error) {
      throw new Error(result.error.message);
    }

    const content1 = result.choices[0].message.content;
    resultJSON.value = JSON.parse(content1);

  } catch (err) {
    error.value = `Error: ${err.message || 'Failed to process file'}`;
    console.error(err);
  } finally {
    loading.value = false;
  }
}

async function extractTextFromPDF(dataURL) {
  return new Promise(async (resolve, reject) => {
    try {
      const pdfjsLib = await import('pdfjs-dist/build/pdf');
      const pdfjsWorker = await import('pdfjs-dist/build/pdf.worker.entry');
      pdfjsLib.GlobalWorkerOptions.workerSrc = pdfjsWorker;

      const base64Data = dataURL.split(',')[1];
      const binaryData = atob(base64Data);
      const length = binaryData.length;
      const dataArray = new Uint8Array(length);
      for (let i = 0; i < length; i++) {
        dataArray[i] = binaryData.charCodeAt(i);
      }

      const pdfDoc = await pdfjsLib.getDocument({ data: dataArray }).promise;
      let fullText = '';

      for (let pageNum = 1; pageNum <= pdfDoc.numPages; pageNum++) {
        const page = await pdfDoc.getPage(pageNum);
        const textContent = await page.getTextContent();
        fullText += textContent.items.map(item => item.str).join(' ') + '\n';
      }

      resolve(fullText);
    } catch (err) {
      reject(new Error('Failed to extract text from PDF'));
    }
  });
}
</script>




