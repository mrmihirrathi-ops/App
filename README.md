# App
........
import { useEffect, useRef, useState } from "react";
import * as THREE from "three";
import { RotateCw, ZoomIn, ZoomOut, Info, ChevronLeft, ChevronRight } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Tooltip, TooltipContent, TooltipTrigger } from "@/components/ui/tooltip";

interface ModelViewerProps {
    modelPath?: string;
    artifactName: string;
    scanImages?: string[];
    identifiedAs?: string;
    // shapeProfile is expected to be an array after being parsed by the API
    shapeProfile?: Array<[number, number]> | string; 
}

// Default profile for safety (a simple rounded shape)
const DEFAULT_PROFILE: Array<[number, number]> = [[0, 0.3], [0.2, 0.7], [0.4, 1], [0.6, 1], [0.8, 0.7], [1, 0.3]];

/**
 * Generates a custom 3D mesh geometry by rotating a 2D profile around the Y-axis.
 * This turns the cross-section data (shapeProfile) into a volumetric object (LatheGeometry).
 */
function generateCustomGeometry(profile: Array<[number, number]>): THREE.BufferGeometry {
    // 1. Convert the profile points into THREE.Vector2 objects.
    // The profile points are [radius, height]. We normalize/scale them for the scene.
    // radius * 1.5 for size, height * 2 - 1.5 to center the shape vertically around y=0.
    const points = profile.map(([radius, height]) => new THREE.Vector2(radius * 1.5, height * 2 - 1.5));
    
    // 2. Use LatheGeometry to create the 3D object by rotating the profile points.
    // 32 segments provide a reasonably smooth surface.
    return new THREE.LatheGeometry(points, 32); 
}

export default function ModelViewer({ modelPath, artifactName, scanImages, identifiedAs, shapeProfile }: ModelViewerProps) {
    const canvasRef = useRef<HTMLDivElement>(null);
    const sceneRef = useRef<THREE.Scene | null>(null);
    const rendererRef = useRef<THREE.WebGLRenderer | null>(null);
    const cameraRef = useRef<THREE.PerspectiveCamera | null>(null);
    const meshRef = useRef<THREE.Mesh | null>(null);
    
    const [currentImageIndex, setCurrentImageIndex] = useState(0);
    const [isRotating, setIsRotating] = useState(false);
    
    // State management for interaction
    const lastMouseRef = useRef({ x: 0, y: 0 });
    const isDraggingRef = useRef(false);
    const rotationStateRef = useRef({ x: 0, y: 0 });

    useEffect(() => {
        if (!canvasRef.current || !scanImages || scanImages.length === 0) return;

        // --- Initialization ---
        
        // Scene setup
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0f0f12);
        sceneRef.current = scene;

        // Camera
        const camera = new THREE.PerspectiveCamera(
            75, 
            canvasRef.current.clientWidth / canvasRef.current.clientHeight, 
            0.1, 
            1000
        );
        camera.position.z = 3;
        cameraRef.current = camera;

        // Renderer
        const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        renderer.setSize(canvasRef.current.clientWidth, canvasRef.current.clientHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        canvasRef.current.appendChild(renderer.domElement);
        rendererRef.current = renderer;

        // Lights
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
        scene.add(ambientLight);
        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
        directionalLight.position.set(5, 5, 5);
        scene.add(directionalLight);

        // --- Texture and Mesh Creation ---

        const atlasCanvas = document.createElement('canvas');
        atlasCanvas.width = 2048;
        atlasCanvas.height = 1024;
        const atlasCtx = atlasCanvas.getContext('2d')!; 
        atlasCtx.fillStyle = '#1a1a1a';
        atlasCtx.fillRect(0, 0, 2048, 1024);

        let imagesLoaded = 0;
        let textureReady = false;

        const createMesh = () => {
            if (meshRef.current) {
                scene.remove(meshRef.current);
            }

            const atlasTexture = new THREE.CanvasTexture(atlasCanvas);
            atlasTexture.magFilter = THREE.LinearFilter;
            atlasTexture.minFilter = THREE.LinearFilter;
            textureReady = true;

            const material = new THREE.MeshPhongMaterial({
                map: atlasTexture,
                emissive: 0x1a1a1a,
                shininess: 30,
                side: THREE.DoubleSide,
            });

            // --- SHAPE FIX: Use shapeProfile for custom geometry (LatheGeometry) ---
            let profilePoints: Array<[number, number]> = DEFAULT_PROFILE;

            // Safely parse shapeProfile if it is a string (though it should be an array from the API)
            if (typeof shapeProfile === 'string') {
                try {
                    profilePoints = JSON.parse(shapeProfile) as Array<[number, number]>;
                } catch (e) {
                    console.error("Using default profile. Failed to parse shapeProfile:", e);
                }
            } else if (Array.isArray(shapeProfile)) {
                profilePoints = shapeProfile;
            }

            const geometry = generateCustomGeometry(profilePoints.length > 0 ? profilePoints : DEFAULT_PROFILE);
            
            const mesh = new THREE.Mesh(geometry, material);
            // LatheGeometry is constructed along the Y-axis. Rotate it 90 degrees
            // on the X-axis so it stands upright on the Z-Y plane.
            mesh.rotation.x = Math.PI / 2; 

            scene.add(mesh);
            meshRef.current = mesh;
            
            // Immediately render the first frame after mesh creation
            renderer.render(scene, camera);
        };

        const addImageToAtlas = (imgSrc: string, index: number) => {
            return new Promise<void>((resolve) => {
                try {
                    const img = new Image();
                    img.crossOrigin = "anonymous";
                    img.onload = () => {
                        // Arrange in 2 rows × 6 columns
                        const cols = 6;
                        const col = index % cols;
                        const row = Math.floor(index / cols);
                        const imgWidth = 2048 / cols;
                        const imgHeight = 1024 / 2;
                        
                        // Draw image onto the atlas canvas
                        atlasCtx.drawImage(img, col * imgWidth, row * imgHeight, imgWidth, imgHeight);
                        
                        imagesLoaded++;
                        if (imagesLoaded === scanImages.length) {
                            createMesh(); // Create the mesh once all textures are loaded
                        }
                        resolve();
                    };
                    img.onerror = () => {
                        // Log error but still count as loaded to ensure createMesh is called
                        console.error(`Failed to load image at index ${index}`);
                        imagesLoaded++;
                        if (imagesLoaded === scanImages.length) {
                            createMesh();
                        }
                        resolve();
                    };
                    img.src = imgSrc;
                } catch (error) {
                    console.error(`Error processing image at index ${index}:`, error);
                    imagesLoaded++;
                    if (imagesLoaded === scanImages.length) {
                        createMesh();
                    }
                    resolve();
                }
            });
        };

        const loadImagesSequentially = async () => {
            // Await all image loadings before attempting to create the texture/mesh
            const promises = scanImages.map((src, idx) => addImageToAtlas(src, idx));
            await Promise.all(promises); 
        };
        
        loadImagesSequentially().catch(console.error);

        // --- Interaction Handlers ---
        
        const onMouseDown = (e: MouseEvent) => {
            isDraggingRef.current = true;
            setIsRotating(true);
            lastMouseRef.current = { x: e.clientX, y: e.clientY };
        };

        const onMouseMove = (e: MouseEvent) => {
            if (!isDraggingRef.current || !meshRef.current) return;
            const deltaX = e.clientX - lastMouseRef.current.x;
            const deltaY = e.clientY - lastMouseRef.current.y;
            
            rotationStateRef.current.y += deltaX * 0.01;
            rotationStateRef.current.x += deltaY * 0.01;
            
            // Clamp X rotation to prevent flipping (looking under/over the object)
            rotationStateRef.current.x = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, rotationStateRef.current.x));
            
            // Note: mesh.rotation.x is the vertical angle, mesh.rotation.z is the spin (due to the PI/2 initial rotation)
            meshRef.current.rotation.x = rotationStateRef.current.x + Math.PI / 2; // Add back the initial alignment rotation
            meshRef.current.rotation.z = rotationStateRef.current.y;
            
            lastMouseRef.current = { x: e.clientX, y: e.clientY };
        };

        const onMouseUp = () => {
            isDraggingRef.current = false;
            setIsRotating(false);
        };

        const onWheel = (e: WheelEvent) => {
            if (canvasRef.current && canvasRef.current.contains(e.target as Node)) {
                e.preventDefault();
                const speed = 0.2;
                const direction = e.deltaY > 0 ? 1 : -1;
                camera.position.z += direction * speed;
                camera.position.z = Math.max(1.5, Math.min(10, camera.position.z));
            }
        };

        renderer.domElement.addEventListener('mousedown', onMouseDown);
        renderer.domElement.addEventListener('mousemove', onMouseMove);
        document.addEventListener('mouseup', onMouseUp);
        renderer.domElement.addEventListener('wheel', onWheel, { passive: false });

        // --- Animation and Resize ---

        const animate = () => {
            requestAnimationFrame(animate);
            // If dragging, rotation is handled by onMouseMove, otherwise auto-rotate
            if (meshRef.current && !isDraggingRef.current && !isRotating) {
                // mesh.rotation.z is the object's primary spin axis after the initial rotation fix
                meshRef.current.rotation.z += 0.005; 
            }
            renderer.render(scene, camera);
        };
        
        animate();

        // Handle resize
        const handleResize = () => {
            if (!canvasRef.current || !rendererRef.current || !cameraRef.current) return;
            const width = canvasRef.current.clientWidth;
            const height = canvasRef.current.clientHeight;
            cameraRef.current.aspect = width / height;
            cameraRef.current.updateProjectionMatrix();
            rendererRef.current.setSize(width, height);
        };
        
        window.addEventListener('resize', handleResize);

        // --- Cleanup ---
        return () => {
            window.removeEventListener('resize', handleResize);
            renderer.domElement.removeEventListener('mousedown', onMouseDown);
            renderer.domElement.removeEventListener('mousemove', onMouseMove);
            document.removeEventListener('mouseup', onMouseUp);
            renderer.domElement.removeEventListener('wheel', onWheel);
            canvasRef.current?.removeChild(renderer.domElement);
            renderer.dispose();
            scene.traverse(object => {
                if (object instanceof THREE.Mesh) {
                    object.geometry.dispose();
                    if (Array.isArray(object.material)) {
                        object.material.forEach(m => m.dispose());
                    } else {
                        object.material.dispose();
                    }
                }
            });
        };
    }, [scanImages, shapeProfile]); // Redraw when images or shapeProfile changes

    const handleResetRotation = () => {
        if (meshRef.current) {
            rotationStateRef.current = { x: 0, y: 0 };
            // Reset to default orientation, accounting for the initial PI/2 alignment
            meshRef.current.rotation.x = Math.PI / 2; 
            meshRef.current.rotation.z = 0; 
        }
    };

    const handlePrevImage = () => {
        if (scanImages && scanImages.length > 0) {
            setCurrentImageIndex((prev) => (prev - 1 + scanImages.length) % scanImages.length);
        }
    };

    const handleNextImage = () => {
        if (scanImages && scanImages.length > 0) {
            setCurrentImageIndex((prev) => (prev + 1) % scanImages.length);
        }
    };

    return (
        <div className="relative aspect-square rounded-lg bg-gradient-to-br from-slate-900 to-slate-950 overflow-hidden border border-slate-800">
            <div ref={canvasRef} className="w-full h-full" style={{ touchAction: 'none' }} />
            
            {/* Help text */}
            <div className="absolute top-4 left-1/2 -translate-x-1/2 text-white text-xs flex items-center gap-2">
                <Tooltip>
                    <TooltipTrigger asChild>
                        <div className="flex items-center gap-2 bg-black/30 backdrop-blur px-2 py-1 rounded">
                            <Info className="h-3 w-3" />
                            <span>Drag to rotate • Scroll to zoom</span>
                        </div>
                    </TooltipTrigger>
                    <TooltipContent side="bottom" className="text-xs max-w-xs">
                        <p>3D model reconstructed from 12 scan images with AI-analyzed shape.</p>
                    </TooltipContent>
                </Tooltip>
            </div>
            
            {/* Artifact identification */}
            {identifiedAs && (
                <div className="absolute top-4 right-4 bg-primary/80 text-white text-xs px-2 py-1 rounded">
                    {identifiedAs}
                </div>
            )}
            
            {/* Scan image navigation */}
            {scanImages && scanImages.length > 0 && (
                <div className="absolute bottom-4 left-4 flex items-center gap-2 bg-black/30 backdrop-blur rounded px-2 py-1">
                    <Button size="icon" variant="ghost" className="h-6 w-6" onClick={handlePrevImage} data-testid="button-prev-image" >
                        <ChevronLeft className="h-3 w-3 text-white" />
                    </Button>
                    <div className="text-white text-xs font-mono">
                        {currentImageIndex + 1}/{scanImages.length}
                    </div>
                    <Button size="icon" variant="ghost" className="h-6 w-6" onClick={handleNextImage} data-testid="button-next-image" >
                        <ChevronRight className="h-3 w-3 text-white" />
                    </Button>
                    <div className="h-10 w-10 ml-2 overflow-hidden rounded-md border border-slate-700">
                        {/* Display current scan image for reference */}
                        <img 
                            src={scanImages[currentImageIndex]} 
                            alt={`Scan ${currentImageIndex + 1}`} 
                            className="w-full h-full object-cover"
                            onError={(e) => { e.currentTarget.src = 'https://placehold.co/100x100/333333/ffffff?text=Err'; }}
                        />
                    </div>
                </div>
            )}
            
            {/* Controls */}
            <div className="absolute bottom-4 right-4 flex gap-2">
                <Tooltip>
                    <TooltipTrigger asChild>
                        <Button size="icon" variant="secondary" onClick={handleResetRotation} data-testid="button-reset-rotation" >
                            <RotateCw className="h-4 w-4" />
                        </Button>
                    </TooltipTrigger>
                    <TooltipContent side="left" className="text-xs">Reset Rotation</TooltipContent>
                </Tooltip>
                <Tooltip>
                    <TooltipTrigger asChild>
                        <Button size="icon" variant="secondary" onClick={() => { if (cameraRef.current) { cameraRef.current.position.z = Math.max(1.5, cameraRef.current.position.z - 0.3); } }} data-testid="button-zoom-in" >
                            <ZoomIn className="h-4 w-4" />
                        </Button>
                    </TooltipTrigger>
                    <TooltipContent side="left" className="text-xs">Zoom In</TooltipContent>
                </Tooltip>
                <Tooltip>
                    <TooltipTrigger asChild>
                        <Button size="icon" variant="secondary" onClick={() => { if (cameraRef.current) { cameraRef.current.position.z = Math.min(10, cameraRef.current.position.z + 0.3); } }} data-testid="button-zoom-out" >
                            <ZoomOut className="h-4 w-4" />
                        </Button>
                    </TooltipTrigger>
                    <TooltipContent side="left" className="text-xs">Zoom Out</TooltipContent>
                </Tooltip>
            </div>
            
            {/* Status indicator */}
            {isRotating && (
                <div className="absolute top-16 right-4 bg-primary/80 text-white text-xs px-2 py-1 rounded">
                    Rotating...
                </div>
            )}
        </div>
    ); 
}
