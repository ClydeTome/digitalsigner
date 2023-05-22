<template>
  <div>
    <input type="file" accept=".pdf" @change="handleFileChange">
    <button @click="undo">Undo</button>
    <button @click="redo">Redo</button>
    <button @click="savePdf">Save PDF</button>
    <div v-for="(page, index) in pages" :key="index">
      <canvas ref="canvas" :id="'canvas' + index" @mousedown="handleMouseDown" @mousemove="handleMouseMove"
        @mouseup="handleMouseUp"></canvas>
    </div>

  </div>
</template>

<script>
import * as pdfjsLib from 'pdfjs-dist/webpack';
import { degrees, PDFDocument, PDFContentStream, PDFOperator, PDFRawStream } from 'pdf-lib';

pdfjsLib.GlobalWorkerOptions.workerSrc = require('pdfjs-dist/build/pdf.worker.js');

export default {
  data() {
    return {
      pdf: null,
      pages: [],
      canvases: [],
      contexts: [],
      originalPdfData: null,
      signatureStart: false,
      currentPath: [], // this will still store the strokes of a single signature
      paths: [], // each index will now correspond to a specific page
      undonePaths: [], // each index will now correspond to a specific page
      currentPageIndex: -1,
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
          this.originalPdfData = pdfData;
          this.loadPdfData(pdfData);
        };
        reader.readAsArrayBuffer(file);
      }
    },
    async loadPdfData(pdfData) {
      const loadingTask = pdfjsLib.getDocument({ data: pdfData });
      const pdf = await loadingTask.promise;
      this.pdf = pdf;

      // Initialize paths and undonePaths arrays here
      this.paths = Array(pdf.numPages).fill(null).map(() => []);
      this.undonePaths = Array(pdf.numPages).fill(null).map(() => []);

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
      this.currentPageIndex = Number(canvas.id.replace('canvas', '')); // get page index from canvas id
    },
    handleMouseMove(event) {
      if (!this.signatureStart) return;

      const canvas = event.target;
      const pageIndex = Number(canvas.id.replace('canvas', '')); // get page index from canvas id

      const rect = canvas.getBoundingClientRect(); // Get the canvas's bounding rectangle

      this.endX = event.clientX - rect.left; // Calculate the relative X position
      this.endY = event.clientY - rect.top; // Calculate the relative Y position


      const context = canvas.getContext('2d');
      context.beginPath();
      context.moveTo(this.startX, this.startY);
      context.lineTo(this.endX, this.endY);
      context.stroke();

      // Push the stroke to currentPath
      this.currentPath.push({ startX: this.startX, startY: this.startY, endX: this.endX, endY: this.endY });

      this.startX = this.endX;
      this.startY = this.endY;
      this.currentPageIndex = Number(canvas.id.replace('canvas', ''));
    },
    handleMouseUp(event) {
      this.signatureStart = false;
      const canvas = event.target;
      const pageIndex = Number(canvas.id.replace('canvas', '')); // get page index from canvas id

      // Signature has ended, push currentPath to the right paths array and clear currentPath
      if (this.currentPath.length) {
        this.paths[pageIndex].push(this.currentPath);
        this.currentPath = [];
      }
      this.currentPageIndex = Number(canvas.id.replace('canvas', ''));
    },
    async savePdf() {
      // Load the original PDF document
      const existingPdfBytes = this.originalPdfData;
      const pdfDoc = await PDFDocument.load(existingPdfBytes);

      // Get the original pages
      const originalPages = pdfDoc.getPages();

      for (let i = 0; i < originalPages.length; i++) {
        console.log(originalPages[i].getSize());
        // Convert the signature data URL to bytes
        const response = await fetch(this.canvases[i].toDataURL('image/png'));
        const blob = await response.blob();

        // Wrap the FileReader in a promise
        const signaturePngBytes = await new Promise((resolve, reject) => {
          const reader = new FileReader();
          reader.onloadend = function () {
            resolve(new Uint8Array(this.result));
          };
          reader.onerror = reject;
          reader.readAsArrayBuffer(blob);
        });

        // Embed the signature image
        const signatureImage = await pdfDoc.embedPng(signaturePngBytes);

        // Get the width and height of the original page
        const size = originalPages[i].getSize();
        const originalPageWidth = size.width;
        const originalPageHeight = size.height;

        // Calculate the signature dimensions
        const signatureDims = signatureImage.scale(1);
        const [signatureWidth, signatureHeight] = [signatureDims.width, signatureDims.height];

        // Calculate the scale for the signature image
        const scale = Math.min(originalPageWidth / signatureWidth, originalPageHeight / signatureHeight);

        // Calculate the position for the signature image
        const xPos = (originalPageWidth - signatureWidth * scale) / 2;
        const yPos = (originalPageHeight - signatureHeight * scale) / 2;

        // Draw the signature image on the page
        originalPages[i].drawImage(signatureImage, {
          x: xPos,
          y: yPos,
          width: signatureWidth * scale,
          height: signatureHeight * scale,
          rotate: degrees(0),
        });
      }

      // Save the PDF with the signatures
      const pdfBytes = await pdfDoc.save();
      const blob = new Blob([pdfBytes], { type: 'application/pdf' });
      const url = window.URL.createObjectURL(blob);
      const link = document.createElement('a');
      link.href = url;
      link.download = 'signed_document.pdf';
      link.click();
    },
    // getContentStream(page) {
    //   return new Promise((resolve, reject) => {
    //     const stream = page.getOperatorList();
    //     const chunks = [];
    //     stream.onData = (chunk) => chunks.push(chunk);
    //     stream.onError = reject;
    //     stream.onEnd = () => resolve(Buffer.concat(chunks));
    //     stream.execute();
    //   });
    // },

    getContentStream(page) {
      return new Promise((resolve, reject) => {
        const stream = page.stream();
        const chunks = [];
        stream.on('data', (chunk) => chunks.push(chunk));
        stream.on('end', () => resolve(Buffer.concat(chunks)));
        stream.on('error', reject);
      });
    },

    insertSignatureOverlay(contentStream, signatureImageRef, xPos, yPos, width, height, rotation) {
      const commands = pdfjsLib.PDFOperator.getCommands(contentStream);

      const signatureCommand = `q ${width} 0 0 ${height} ${xPos} ${yPos} cm /${signatureImageRef} Do Q`;
      const rotationCommand = `q ${Math.cos(rotation)} ${Math.sin(rotation)} ${-Math.sin(rotation)} ${Math.cos(rotation)} 0 0 cm`;

      const modifiedCommands = [rotationCommand, signatureCommand, 'Q'];

      const modifiedContentStream = [...commands, ...modifiedCommands].join('\n');
      return modifiedContentStream;
    },
    undo() {
      // We can't undo if there are no pages or no page has been interacted with yet
      if (!this.pages.length || this.currentPageIndex < 0) return;

      // Use the index of the last interacted page
      const pageIndex = this.currentPageIndex;

      // If there are no paths for this page, nothing to undo
      if (!this.paths[pageIndex].length) return;

      const pathToUndo = this.paths[pageIndex].pop();
      this.undonePaths[pageIndex].push(pathToUndo);

      // Redraw the specific canvas
      this.redrawCanvas(pageIndex);
    },

    redo() {
      // We can't redo if there are no pages or no page has been interacted with yet
      if (!this.pages.length || this.currentPageIndex < 0) return;

      // Use the index of the last interacted page
      const pageIndex = this.currentPageIndex;

      // If there are no undone paths for this page, nothing to redo
      if (!this.undonePaths[pageIndex].length) return;

      const pathToRedo = this.undonePaths[pageIndex].pop();
      this.paths[pageIndex].push(pathToRedo);

      // Redraw the specific canvas
      this.redrawCanvas(pageIndex);
    },

    async redrawCanvas(pageIndex) {
      const canvas = this.canvases[pageIndex];
      const context = this.contexts[pageIndex];

      // Clear the canvas
      context.clearRect(0, 0, canvas.width, canvas.height);

      // Re-render the PDF page on the canvas
      await this.renderPdfOnCanvas(this.pages[pageIndex], pageIndex);

      // Redraw the paths for this page
      this.paths[pageIndex].forEach((path) => { // path is now an entire signature
        path.forEach((stroke) => { // iterate over each stroke within the signature
          context.beginPath();
          context.moveTo(stroke.startX, stroke.startY);
          context.lineTo(stroke.endX, stroke.endY);
          context.stroke();
        });
      });
    },


  },
};
</script>


