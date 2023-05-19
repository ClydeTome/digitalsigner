<template>
  <div>
    <input type="file" accept=".pdf" @change="handleFileChange">
    <div v-for="(page, index) in pages" :key="index">
      <canvas ref="canvas" :id="'canvas' + index" @mousedown="handleMouseDown" @mousemove="handleMouseMove"
        @mouseup="handleMouseUp"></canvas>
    </div>
    <button @click="savePdf">Save PDF</button>
  </div>
</template>

<script>
import * as pdfjsLib from 'pdfjs-dist/webpack';
import { PDFDocument } from 'pdf-lib';

pdfjsLib.GlobalWorkerOptions.workerSrc = require('pdfjs-dist/build/pdf.worker.js');

export default {
  data() {
    return {
      pdf: null,
      pages: [],
      canvases: [],
      contexts: [],
      signatureStart: false,
      startX: 0,
      startY: 0,
      endX: 0,
      endY: 0,
    };
  },
  methods: {
    async handleFileChange(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = () => {
          const pdfData = new Uint8Array(reader.result);
          this.loadPdfData(pdfData);
        };
        reader.readAsArrayBuffer(file);
      }
    },
    async loadPdfData(pdfData) {
      const loadingTask = pdfjsLib.getDocument({ data: pdfData });
      const pdf = await loadingTask.promise;
      this.pdf = pdf;
      for (let i = 1; i <= pdf.numPages; i++) {
        const page = await this.pdf.getPage(i);
        this.pages.push(page);
      }

      this.$nextTick(async () => {
        for (let i = 0; i < this.pages.length; i++) {
          this.canvases[i] = this.$refs.canvas[i];
          this.contexts[i] = this.canvases[i].getContext('2d');
          await this.renderPdfOnCanvas(this.pages[i], i);
        }
      });
    },
    async renderPdfOnCanvas(page, index) {
      const canvas = this.canvases[index];
      const context = this.contexts[index];
      const viewport = await page.getViewport({ scale: 2 });
      canvas.width = viewport.width;
      canvas.height = viewport.height;

      const renderContext = {
        canvasContext: context,
        viewport: viewport,
      };

      await page.render(renderContext).promise;
    },
    handleMouseDown(event) {
      this.signatureStart = true;
      const canvas = event.target;
      const rect = canvas.getBoundingClientRect(); // Get the canvas's bounding rectangle
      this.startX = event.clientX - rect.left; // Calculate the relative X position
      this.startY = event.clientY - rect.top; // Calculate the relative Y position
    },
    handleMouseMove(event) {
      if (!this.signatureStart) return;

      const canvas = event.target;
      const rect = canvas.getBoundingClientRect(); // Get the canvas's bounding rectangle
      this.endX = event.clientX - rect.left; // Calculate the relative X position
      this.endY = event.clientY - rect.top; // Calculate the relative Y position

      const context = canvas.getContext('2d');
      context.beginPath();
      context.moveTo(this.startX, this.startY);
      context.lineTo(this.endX, this.endY);
      context.stroke();

      this.startX = this.endX;
      this.startY = this.endY;
    },
    handleMouseUp() {
      this.signatureStart = false;
    },
    async savePdf() {
      const pdfDoc = await PDFDocument.create();
      for (let i = 0; i < this.pages.length; i++) {
        const canvas = this.canvases[i];
        const [width, height] = [canvas.width, canvas.height];
        const newPage = pdfDoc.addPage([width, height]);

        const pngUrl = canvas.toDataURL('image/png');
        const response = await fetch(pngUrl);
        const pngBlob = await response.blob();
        const pngBuffer = await new Response(pngBlob).arrayBuffer();
        const pngUint8Array = new Uint8Array(pngBuffer);

        const signatureImage = await pdfDoc.embedPng(pngUint8Array);
        newPage.drawImage(signatureImage, {
          x: 0,
          y: 0,
          width: width,
          height: height,
        });
      }

      const pdfBytes = await pdfDoc.save();
      const blob = new Blob([pdfBytes], { type: 'application/pdf' });
      const url = window.URL.createObjectURL(blob);
      const link = document.createElement('a');
      link.href = url;
      link.download = 'signed_document.pdf';
      link.click();
    },
  },
};
</script>


