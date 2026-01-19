<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Archaeology Digital Twin Viewer</title>
    <script type="module" src="https://ajax.googleapis.com/ajax/libs/model-viewer/3.4.0/model-viewer.min.js"></script>
    <style>
        body { margin: 0; font-family: sans-serif; background: #1a1a1a; color: white; }
        model-viewer { width: 100%; height: 80vh; background-color: #222; }
        .controls { padding: 20px; text-align: center; }
        button { 
            background: #d4af37; border: none; padding: 15px 30px; 
            font-size: 18px; border-radius: 5px; cursor: pointer; color: black; font-weight: bold;
        }
        h1 { margin: 10px; font-size: 24px; }
    </style>
</head>
<body>

    <h1>Artifact Digital Twin</h1>
    
    <model-viewer 
        src="artifact.glb" 
        ar 
        camera-controls 
        shadow-intensity="1" 
        exposure="1.2"
        auto-rotate>
    </model-viewer>

    <div class="controls">
        <p>Archaeologists: Use the button below to log data for this find.</p>
        <button onclick="window.location.href='YOUR_GOOGLE_FORM_LINK_HERE'">Fill Artifact Data</button>
    </div>

</body>
</html>
