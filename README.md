
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Tower Defense</title>
<style>
body {
  margin: 0;
  background: #172033;
  color: white;
  font-family: Arial, sans-serif;
  text-align: center;
}
h1 { color: #ffe080; }
#info { padding: 10px; font-size: 18px; }
button {
  padding: 10px;
  margin: 4px;
  border: 0;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
}
canvas {
  width: 95%;
  max-width: 900px;
  background: #8acb70;
  border: 3px solid #fff;
  border-radius: 10px;
  touch-action: manipulation;
}
.tower { background: #344966; color: white; }
.action { background: #ffe080; color: #172033; }
</style>
</head>
<body>

<h1>🛡️ Tower Defense</h1>
<div id="info">
❤️ Vidas: <span id="vidas">20</span> |
🪙 Ouro: <span id="ouro">180</span> |
🌊 Onda: <span id="onda">0</span>
</div>

<div>
  <button class="tower" onclick="escolher(0)">🏹 Atirador - 60</button>
  <button class="tower" onclick="escolher(1)">💣 Canhão - 95</button>
  <button class="tower" onclick="escolher(2)">🧙 Mago - 115</button>
  <button class="tower" onclick="escolher(3)">❄️ Gelo - 85</button>
</div>

<canvas id="jogo" width="900" height="570"></canvas>
<br>
<button class="action" onclick="iniciarOnda()">Começar onda</button>
<button class="action" onclick="reiniciar()">Reiniciar</button>
<p id="mensagem">Escolha um boneco e clique na grama!</p>

<script>
const canvas = document.getElementById("jogo");
const ctx = canvas.getContext("2d");

const caminho = [
  [-20,105],[150,105],[150,245],[365,245],
  [365,105],[590,105],[590,390],
  [760,390],[760,500],[920,500]
];

const tipos = [
  {nome:"Atirador",emoji:"🏹",custo:60,
   dano:13,alcance:125,velocidade:0.65,cor:"#4b91e2"},
  {nome:"Canhão",emoji:"💣",custo:95,
   dano:36,alcance:105,velocidade:1.35,cor:"#d87545"},
  {nome:"Mago",emoji:"🧙",custo:115,
   dano:19,alcance:145,velocidade:0.9,cor:"#a66ce8"},
  {nome:"Gelo",emoji:"❄️",custo:85,
   dano:8,alcance:115,velocidade:0.55,cor:"#55cbd8",
   gelo:true}
];

let bonecos = [];
let monstros = [];
let tiros = [];
let ouro = 180;
let vidas = 20;
let onda = 0;
let tipoEscolhido = 0;
let jogando = false;
let fim = false;
let faltam = 0;
let relogio = 0;
let ultimoTempo = 0;

function escolher(tipo) {
  tipoEscolhido = tipo;
  document.getElementById("mensagem").textContent =
    "Selecionado: " + tipos[tipo].nome +
    ". Clique na grama para colocar!";
}

function distancia(a,b) {
  return Math.hypot(a.x-b.x,a.y-b.y);
}

function posicao(dist) {
  for (let i=1; i<caminho.length; i++) {
    const a=caminho[i-1], b=caminho[i];
    const tamanho=Math.hypot(b[0]-a[0],b[1]-a[1]);

    if (dist <= tamanho) {
      const t=dist/tamanho;
      return {
        x:a[0]+(b[0]-a[0])*t,
        y:a[1]+(b[1]-a[1])*t
      };
    }
    dist-=tamanho;
  }
  return {x:920,y:500};
}

function comprimentoCaminho() {
  let total=0;
  for(let i=1;i<caminho.length;i++) {
    total+=Math.hypot(
      caminho[i][0]-caminho[i-1][0],
      caminho[i][1]-caminho[i-1][1]
    );
  }
  return total;
}
const caminhoTotal=comprimentoCaminho();

function naEstrada(x,y) {
  for(let i=1;i<caminho.length;i++) {
    const a=caminho[i-1],b=caminho[i];
    const dx=b[0]-a[0],dy=b[1]-a[1];
    const t=Math.max(0,Math.min(1,
      ((x-a[0])*dx+(y-a[1])*dy)/(dx*dx+dy*dy)
    ));
    const px=a[0]+t*dx,py=a[1]+t*dy;
    if(Math.hypot(x-px,y-py)<35) return true;
  }
  return false;
}

canvas.addEventListener("click", e => {
  if(fim) return;

  const r=canvas.getBoundingClientRect();
  const x=(e.clientX-r.left)*canvas.width/r.width;
  const y=(e.clientY-r.top)*canvas.height/r.height;

  if(naEstrada(x,y)) {
    document.getElementById("mensagem").textContent =
      "Coloque os bonecos fora da estrada!";
    return;
  }

  if(bonecos.some(b=>Math.hypot(b.x-x,b.y-y)<34)) {
    document.getElementById("mensagem").textContent =
      "Deixe espaço entre os bonecos!";
    return;
  }

  const t=tipos[tipoEscolhido];
  if(ouro<t.custo) {
    document.getElementById("mensagem").textContent =
      "Você não tem ouro suficiente!";
    return;
  }

  bonecos.push({
    x,y,tipo:tipoEscolhido,
    dano:t.dano,alcance:t.alcance,
    espera:0,nivel:1
  });

  ouro-=t.custo;
  atualizarInfo();
});

function iniciarOnda() {
  if(jogando || fim) return;
  onda++;
  faltam=7+onda*3;
  relogio=0;
  jogando=true;
  document.getElementById("mensagem").textContent =
    "Onda "+onda+" começou! Defenda a base!";
  atualizarInfo();
}

function criarMonstro() {
  const p=posicao(0);
  const vida=34+onda*14;
  monstros.push({
    x:p.x,y:p.y,dist:0,
    vida,maxVida:vida,
    velocidade:36+Math.min(onda*1.6,22),
    lento:0, vivo:true
  });
}

function atualizar(dt) {
  if(fim) return;

  if(jogando) {
    relogio-=dt;
    if(faltam>0 && relogio<=0) {
      criarMonstro();
      faltam--;
      relogio=Math.max(0.32,0.9-onda*0.025);
    }
  }

  for(const m of monstros) {
    if(!m.vivo) continue;

    if(m.lento>0) m.lento-=dt;
    m.dist+=m.velocidade*(m.lento>0?0.5:1)*dt;

    const p=posicao(m.dist);
    m.x=p.x;
    m.y=p.y;

    if(m.dist>=caminhoTotal) {
      m.vivo=false;
      vidas--;

      if(vidas<=0) {
        vidas=0;
        fim=true;
        jogando=false;
        document.getElementById("mensagem").textContent =
          "Fim de jogo! Clique em Reiniciar.";
      }
    }
  }

  for(const b of bonecos) {
    b.espera-=dt;
    if(b.espera>0) continue;

    const t=tipos[b.tipo];
    const alvo=monstros
      .filter(m=>m.vivo && distancia(b,m)<=b.alcance)
      .sort((a,c)=>c.dist-a.dist)[0];

    if(alvo) {
      tiros.push({
        x:b.x,y:b.y,alvo,
        dano:b.dano,
        cor:t.cor,
        gelo:!!t.gelo
      });
      b.espera=t.velocidade;
    }
  }

  for(const tiro of tiros) {
    if(!tiro.alvo.vivo) {
      tiro.pronto=true;
      continue;
    }

    const a=tiro.alvo;
    const dx=a.x-tiro.x,dy=a.y-tiro.y;
    const d=Math.hypot(dx,dy);
    const passo=420*dt;

    if(d<passo+8) {
      a.vida-=tiro.dano;
      if(tiro.gelo) a.lento=1.8;
      tiro.pronto=true;

      if(a.vida<=0 && a.vivo) {
        a.vivo=false;
        ouro+=8+Math.floor(onda/2);
      }
    } else {
      tiro.x+=dx/d*passo;
      tiro.y+=dy/d*passo;
    }
  }

  tiros=tiros.filter(t=>!t.pronto);
  monstros=monstros.filter(m=>m.vivo);

  if(jogando && faltam===0 && monstros.length===0) {
    jogando=false;
    ouro+=45+onda*5;
    document.getElementById("mensagem").textContent =
      "Onda concluída! Você ganhou "+(45+onda*5)+" de ouro!";
  }

  atualizarInfo();
}

function atualizarInfo() {
  document.getElementById("vidas").textContent=vidas;
  document.getElementById("ouro").textContent=ouro;
  document.getElementById("onda").textContent=onda;
}

function desenhar() {
  ctx.clearRect(0,0,900,570);
  ctx.fillStyle="#8acb70";
  ctx.fillRect(0,0,900,570);

  ctx.strokeStyle="rgba(37,94,52,.18)";
  for(let x=0;x<900;x+=30) {
    ctx.beginPath();ctx.moveTo(x,0);ctx.lineTo(x,570);ctx.stroke();
  }
  for(let y=0;y<570;y+=30) {
    ctx.beginPath();ctx.moveTo(0,y);ctx.lineTo(900,y);ctx.stroke();
  }

  ctx.lineCap="round";
  ctx.lineJoin="round";
  ctx.strokeStyle="#c9b58b";
  ctx.lineWidth=72;
  ctx.beginPath();
  caminho.forEach((p,i)=>i
    ?ctx.lineTo(p[0],p[1])
    :ctx.moveTo(p[0],p[1]));
  ctx.stroke();

  ctx.strokeStyle="#e6d4ae";
  ctx.lineWidth=62;
  ctx.stroke();

  ctx.font="28px sans-serif";
  ctx.fillText("🏰",832,500);

  for(const b of bonecos) {
    const t=tipos[b.tipo];
    ctx.beginPath();
    ctx.arc(b.x,b.y,22,0,Math.PI*2);
    ctx.fillStyle=t.cor;
    ctx.fill();
    ctx.strokeStyle="#243047";
    ctx.lineWidth=3;
    ctx.stroke();

    ctx.font="24px sans-serif";
    ctx.textAlign="center";
    ctx.fillText(t.emoji,b.x,b.y+8);

    ctx.font="11px sans-serif";
    ctx.fillStyle="white";
    ctx.fillText("Nv."+b.nivel,b.x,b.y+34);
  }

  for(const m of monstros) {
    ctx.beginPath();
    ctx.arc(m.x,m.y,13,0,Math.PI*2);
    ctx.fillStyle=m.lento>0?"#79e7f3":"#d94f63";
    ctx.fill();
    ctx.strokeStyle="#7d2435";
    ctx.lineWidth=2;
    ctx.stroke();

    ctx.fillStyle="#263247";
    ctx.fillRect(m.x-16,m.y-22,32,5);
    ctx.fillStyle="#54df72";
    ctx.fillRect(m.x-16,m.y-22,
      32*Math.max(0,m.vida/m.maxVida),5);

    ctx.font="18px sans-serif";
    ctx.textAlign="center";
    ctx.fillText("👾",m.x,m.y+6);
  }

  for(const t of tiros) {
    ctx.beginPath();
    ctx.arc(t.x,t.y,5,0,Math.PI*2);
    ctx.fillStyle=t.cor;
    ctx.fill();
  }

  ctx.textAlign="left";

  if(fim) {
    ctx.fillStyle="rgba(10,18,30,.75)";
    ctx.fillRect(0,0,900,570);
    ctx.fillStyle="white";
    ctx.textAlign="center";
    ctx.font="bold 38px sans-serif";
    ctx.fillText("FIM DE JOGO",450,270);
  }
}

function reiniciar() {
  bonecos=[];
  monstros=[];
  tiros=[];
  ouro=180;
  vidas=20;
  onda=0;
  jogando=false;
  fim=false;
  faltam=0;
  atualizarInfo();
  document.getElementById("mensagem").textContent =
    "Escolha um boneco e comece a jogar!";
}

function loop(tempo) {
  const dt=Math.min((tempo-ultimoTempo)/1000||0,0.05);
  ultimoTempo=tempo;
  atualizar(dt);
  desenhar();
  requestAnimationFrame(loop);
}

atualizarInfo();
requestAnimationFrame(loop);
</script>
</body>
</html>
