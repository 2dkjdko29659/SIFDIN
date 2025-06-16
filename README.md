# SIFDIN<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>AI Photo Editor</title>
<style>
  body { font-family: Arial, sans-serif; text-align: center; padding: 10px; background: #f3f3f3; }
  #canvas-container { border: 1px solid #ccc; margin: 20px auto; width: 600px; height: 400px; background: white; }
  .toolbar { margin: 10px auto; }
  button, input[type=range] { margin: 5px; }
</style>
</head>
<body>

<h1>AI Photo Editor</h1>

<input type="file" id="upload" accept="image/*" />
<div id="canvas-container">
  <canvas id="canvas" width="600" height="400"></canvas>
</div>

<div class="toolbar">
  <button onclick="rotateImage(90)">Rotate 90°</button>
  <button onclick="flipImage('horizontal')">Flip Horizontal</button>
  <button onclick="flipImage('vertical')">Flip Vertical</button>

  <label>Brightness <input type="range" id="brightness" min="-1" max="1" step="0.01" value="0" oninput="applyFilters()" /></label>
  <label>Contrast <input type="range" id="contrast" min="-1" max="1" step="0.01" value="0" oninput="applyFilters()" /></label>
  <label>Blur <input type="range" id="blur" min="0" max="10" step="0.1" value="0" oninput="applyFilters()" /></label>

  <button onclick="removeBackground()">Remove Background (AI)</button>
  <button onclick="saveImage()">Save Image</button>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/fabric.js/5.2.4/fabric.min.js"></script>
<script>
  const canvas = new fabric.Canvas('canvas');
  let imgInstance = null;

  document.getElementById('upload').addEventListener('change', function(e) {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = function(f) {
      fabric.Image.fromURL(f.target.result, function(img) {
        if(imgInstance) {
          canvas.remove(imgInstance);
        }
        imgInstance = img;
        imgInstance.set({ left: 0, top: 0, angle: 0 });
        imgInstance.scaleToWidth(canvas.width);
        imgInstance.scaleToHeight(canvas.height);
        canvas.add(imgInstance);
        canvas.renderAll();
        resetFilters();
      });
    };
    reader.readAsDataURL(file);
  });

  function rotateImage(angle) {
    if(!imgInstance) return;
    imgInstance.rotate((imgInstance.angle + angle) % 360);
    canvas.renderAll();
  }

  function flipImage(direction) {
    if(!imgInstance) return;
    if(direction === 'horizontal') {
      imgInstance.toggle('flipX');
    } else {
      imgInstance.toggle('flipY');
    }
    canvas.renderAll();
  }

  function applyFilters() {
    if(!imgInstance) return;
    const brightness = parseFloat(document.getElementById('brightness').value);
    const contrast = parseFloat(document.getElementById('contrast').value);
    const blur = parseFloat(document.getElementById('blur').value);

    const filters = [];
    filters.push(new fabric.Image.filters.Brightness({brightness}));
    filters.push(new fabric.Image.filters.Contrast({contrast}));
    if (blur > 0) filters.push(new fabric.Image.filters.Blur({blur}));

    imgInstance.filters = filters;
    imgInstance.applyFilters();
    canvas.renderAll();
  }

  function resetFilters() {
    document.getElementById('brightness').value = 0;
    document.getElementById('contrast').value = 0;
    document.getElementById('blur').value = 0;
    if(!imgInstance) return;
    imgInstance.filters = [];
    imgInstance.applyFilters();
    canvas.renderAll();
  }

  async function removeBackground() {
    if (!imgInstance) return alert('Please upload an image first.');

    // Get image as base64
    const imgDataUrl = canvas.toDataURL({format: 'png'});

    // Use remove.bg API (example)
    // You need to get your own API key from https://www.remove.bg/api
    const apiKey = 'YOUR_REMOVE_BG_API_KEY'; // <-- ضع مفتاح API الخاص بك هنا

    const response = await fetch('https://api.remove.bg/v1.0/removebg', {
      method: 'POST',
      headers: {
        'X-Api-Key': apiKey,
      },
      body: createFormData(imgDataUrl),
    });

    if (response.ok) {
      const blob = await response.blob();
      const url = URL.createObjectURL(blob);
      fabric.Image.fromURL(url, function(img) {
        canvas.remove(imgInstance);
        imgInstance = img;
        imgInstance.set({ left: 0, top: 0 });
        imgInstance.scaleToWidth(canvas.width);
        imgInstance.scaleToHeight(canvas.height);
        canvas.add(imgInstance);
        canvas.renderAll();
        resetFilters();
      });
    } else {
      const err = await response.json();
      alert('Error removing background: ' + err.errors[0].title);
    }
  }

  function createFormData(imageBase64) {
    const fd = new FormData();
    // remove.bg API requires the file in binary - here we convert base64 to binary
    const blob = dataURItoBlob(imageBase64);
    fd.append('image_file', blob, 'image.png');
    fd.append('size', 'auto');
    return fd;
  }

  // Helper function to convert base64 to blob
  function dataURItoBlob(dataURI) {
    const byteString = atob(dataURI.split(',')[1]);
    const mimeString = dataURI.split(',')[0].split(':')[1].split(';')[0];
    const ab = new ArrayBuffer(byteString.length);
    const ia = new Uint8Array(ab);
    for (let i = 0; i < byteString.length; i++) ia[i] = byteString.charCodeAt(i);
    return new Blob([ab], {type: mimeString});
  }

  function saveImage() {
    if (!imgInstance) return alert('No image to save!');
    const dataURL = canvas.toDataURL({format: 'png'});
    const link = document.createElement('a');
    link.href = dataURL;
    link.download = 'edited_image.png';
    link.click();
  }
</script>

</body>
</html>
