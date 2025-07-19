# app.py  
import uvicorn, json, random, math  
from fastapi import FastAPI, Request  
from fastapi.staticfiles import StaticFiles  
from fastapi.templating import Jinja2Templates  
from fastapi.responses import HTMLResponse  
from pathlib import Path  
  
app = FastAPI(title="Pokémon Fusion Lab")  
templates = Jinja2Templates(directory="templates")  
(Path("templates")/Path("static")).mkdir(parents=True, exist_ok=True)  
  
CDN = "https://raw.githubusercontent.com/kleampa/pokemon-3d/main/glb"  
  
# generate 100 dummy Pokémon  
POKEMON = [  
    dict(  
        id=i,  
        name=f"Pokemon-{i}",  
        url=f"{CDN}/{i}.glb",  
        color=random.randint(0x111111, 0xFFFFFF),  
        stats=dict(hp=40+i%60, atk=40+(i*7)%60, def=40+(i*5)%60, spd=40+(i*3)%60)  
    )  
    for i in range(1, 101)  
]  
  
def fuse(base, donor, seed=0):  
    """Deterministic random fusion"""  
    rnd = lambda: (seed:=seed+1, (seed*9301+49297)%233280/233280)[1]  
    color = base["color"] ^ donor["color"]  
    stats = {k:int((base["stats"][k]*.6+donor["stats"][k]*.4)+rnd()*10) for k in base["stats"]}  
    return dict(  
        id=f"{base['id']}-{donor['id']}-{seed}",  
        name=base["name"][:3]+donor["name"][-3:]+f"-{seed}",  
        url=base["url"],      # body  
        parts=donor["url"],   # parts  
        color=color,  
        stats=stats,  
        seed=seed  
    )  
  
@app.get("/", response_class=HTMLResponse)  
async def root(request: Request):  
    return templates.TemplateResponse("index.html", {"request":request, "pokemon":POKEMON})  
  
@app.get("/api/fuse")  
async def api_fuse(base:int, donor:int, seed:int=0):  
    return fuse(POKEMON[base-1], POKEMON[donor-1], seed)  
  
# serve the single template  
templates.env.globals.update({"json":json})  
  
# write template inline  
(Path("templates")/"index.html").write_text('''  
<!DOCTYPE html>  
<html lang="en">  
<head>  
  <meta charset="UTF-8"/>  
  <title>Pokémon Fusion Lab</title>  
  <script src="https://cdn.tailwindcss.com"></script>  
  <script src="https://unpkg.com/three@0.158.0/build/three.min.js"></script>  
  <script src="https://unpkg.com/three@0.158.0/examples/js/loaders/GLTFLoader.js"></script>  
  <script src="https://unpkg.com/three@0.158.0/examples/js/controls/OrbitControls.js"></script>  
  <style>  
    body{background:#000;color:#00ff00;font-family:monospace}  
    .btn{border:2px solid #00ff00;padding:.5rem 1rem;border-radius:.5rem;cursor:pointer}  
    .btn:hover{background:#00ff30;color:#000}  
  </style>  
</head>  
<body class="min-h-screen flex flex-col items-center">  
  <h1 class="text-3xl mt-4">Pokémon Fusion Lab</h1>  
  
  <!-- Fuse button -->  
  <button id="fuseBtn" class="btn fixed top-4 left-1/2 -translate-x-1/2 z-20">Fuse</button>  
  
  <!-- Modal -->  
  <div id="modal" class="hidden fixed inset-0 bg-black/80 flex items-center justify-center z-30">  
    <div class="bg-neutral-900 border border-green-400 rounded-xl p-6 w-full max-w-4xl">  
      <h2 class="text-xl mb-4">Select two Pokémon</h2>  
      <div class="grid md:grid-cols-2 gap-4">  
        <div>  
          <p>Base Body (Slot-1)</p>  
          <select id="baseSel" class="w-full bg-black border border-green-400 mt-1">  
            {% for p in pokemon %}<option value="{{p.id}}">{{p.name}}</option>{% endfor %}  
          </select>  
        </div>  
        <div>  
          <p>Donor Parts (Slot-2)</p>  
          <select id="donorSel" class="w-full bg-black border border-green-400 mt-1">  
            {% for p in pokemon %}<option value="{{p.id}}">{{p.name}}</option>{% endfor %}  
          </select>  
        </div>  
      </div>  
      <div class="flex gap-4 justify-center mt-6">  
        <button id="createBtn" class="btn">Create</button>  
        <button id="recreateBtn" class="btn">Recreate</button>  
        <button id="closeBtn" class="btn">Close</button>  
      </div>  
    </div>  
  </div>  
  
  <!-- 3D viewer -->  
  <div id="viewer" class="w-full h-96 md:h-[600px] mt-32"></div>  
  
  <!-- info -->  
  <div id="info" class="mt-4 text-center"></div>  
  
  <script>  
  const pokemon = {{ pokemon|json }};  
  let fusion=null, seed=0;  
  const scene=new THREE.Scene();  
  const cam=new THREE.PerspectiveCamera(45,innerWidth/innerHeight,0.1,100);  
  cam.position.set(0,1,3);  
  const renderer=new THREE.WebGLRenderer({alpha:true});  
  renderer.setSize(window.innerWidth, window.innerHeight/2);  
  document.getElementById('viewer').appendChild(renderer.domElement);  
  const ctrl=new THREE.OrbitControls(cam,renderer.domElement);  
  const light=new THREE.AmbientLight(0xffffff,0.6);  
  scene.add(light);  
  scene.add(new THREE.DirectionalLight(0x00ff00,1).position.set(5,5,5));  
  
  let model=null;  
  function load(url){  
    if(model) scene.remove(model);  
    const loader=new THREE.GLTFLoader();  
    loader.load(url,gltf=>{  
      model=gltf.scene;  
      model.scale.set(1.2,1.2,1.2);  
      scene.add(model);  
    });  
  }  
  
  async function createFusion(){  
    const b=document.getElementById('baseSel').value;  
    const d=document.getElementById('donorSel').value;  
    const res=await fetch(/api/fuse?base=${b}&donor=${d}&seed=${seed});  
    fusion=await res.json();  
    load(fusion.url);  
    document.getElementById('info').innerHTML=  
      <h2 class="text-2xl">${fusion.name}</h2>  
      <div class="grid grid-cols-2 gap-2 mt-2">${Object.entries(fusion.stats).map(([k,v])=><span>${k}:${v}</span>).join('')}</div>  
    ;  
  }  
  
  document.getElementById('fuseBtn').onclick=()=>document.getElementById('modal').classList.remove('hidden');  
  document.getElementById('closeBtn').onclick=()=>document.getElementById('modal').classList.add('hidden');  
  document.getElementById('createBtn').onclick=()=>{seed=0; createFusion(); document.getElementById('modal').classList.add('hidden');};  
  document.getElementById('recreateBtn').onclick=()=>{seed+=1; createFusion();};  
  
  function animate(){  
    requestAnimationFrame(animate);  
    if(model) model.rotation.y+=0.005;  
    renderer.render(scene,cam);  
  }  
  animate();  
  window.addEventListener('resize',()=>{  
    renderer.setSize(window.innerWidth, window.innerHeight/2);  
    cam.aspect=innerWidth/(innerHeight/2); cam.updateProjectionMatrix();  
  });  
  </script>  
</body>  
</html>  
''')  
  
if __name__ == "__main__":  
    uvicorn.run(app, host="0.0.0.0", port=8000)  
