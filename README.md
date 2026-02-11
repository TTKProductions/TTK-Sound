<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>TTK Sound | Ultimate Studio</title>
    <style>
        :root { --bg: #050505; --accent: #32ff7e; --glass: rgba(255,255,255,0.05); }
        body { background: var(--bg); color: white; font-family: sans-serif; margin: 0; display: grid; grid-template-columns: 240px 1fr; height: 100vh; overflow: hidden; }
        nav { background: #000; padding: 25px; border-right: 1px solid #222; }
        .logo { font-size: 22px; font-weight: 900; color: var(--accent); margin-bottom: 30px; }
        .nav-btn { padding: 12px; cursor: pointer; border-radius: 8px; transition: 0.2s; display: block; color: #aaa; text-decoration: none; }
        .nav-btn:hover { background: var(--glass); color: var(--accent); }
        main { padding: 40px; overflow-y: auto; padding-bottom: 120px; }
        .grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 20px; }
        .card { background: #111; padding: 15px; border-radius: 12px; position: relative; border: 1px solid #222; transition: 0.3s; }
        .card:hover { border-color: var(--accent); transform: translateY(-3px); }
        .card img { width: 100%; aspect-ratio: 1; border-radius: 8px; object-fit: cover; }
        .stats { font-size: 12px; color: var(--accent); margin-top: 8px; display: flex; justify-content: space-between; }
        #upload-modal { display: none; position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); background: #151515; padding: 30px; border-radius: 20px; border: 2px solid var(--accent); z-index: 1000; width: 400px; }
        input { width: 100%; padding: 10px; margin: 10px 0; background: #000; border: 1px solid #333; color: white; border-radius: 5px; }
        footer { position: fixed; bottom: 0; width: 100%; height: 100px; background: rgba(0,0,0,0.95); backdrop-filter: blur(15px); border-top: 1px solid #222; display: flex; align-items: center; padding: 0 30px; box-sizing: border-box; }
        .p-info { width: 30%; display: flex; align-items: center; gap: 15px; }
        .p-info img { width: 60px; height: 60px; border-radius: 5px; }
        .controls { flex: 1; display: flex; flex-direction: column; align-items: center; gap: 10px; }
        .btn-row { display: flex; align-items: center; gap: 20px; font-size: 20px; cursor: pointer; }
        .progress-bar { width: 100%; height: 5px; background: #333; border-radius: 10px; cursor: pointer; position: relative; }
        #progress-fill { height: 100%; background: var(--accent); width: 0%; border-radius: 10px; }
        .vol-row { width: 20%; display: flex; align-items: center; gap: 10px; justify-content: flex-end; }
    </style>
</head>
<body>

<nav>
    <div class="logo">TTK SOUND</div>
    <div class="nav-btn" onclick="showUpload()">📤 Upload Track</div>
    <div class="nav-btn" onclick="saveAll()">💾 Save Platform</div>
</nav>

<main>
    <h1 id="welcome">TTK Studio</h1>
    <div class="grid" id="grid"></div>
</main>

<div id="upload-modal">
    <h3>Studio Upload</h3>
    <input type="text" id="up-title" placeholder="Song Title">
    <input type="text" id="up-artist" placeholder="Artist Name">
    <label>Select Audio:</label>
    <input type="file" id="up-audio" accept="audio/*">
    <label>Select Cover Art (No AI):</label>
    <input type="file" id="up-image" accept="image/*">
    <button onclick="processUpload()" style="background: var(--accent); border:none; padding:10px; width:100%; font-weight:bold; cursor:pointer;">PUBLISH TO TTK</button>
    <button onclick="this.parentElement.style.display='none'" style="background:none; color:white; border:none; width:100%; margin-top:10px; cursor:pointer;">Cancel</button>
</div>

<footer>
    <div class="p-info">
        <img src="https://via.placeholder.com/60" id="now-img">
        <div><b id="now-title">Idle</b><br><small id="now-artist">TTK Sound</small></div>
    </div>
    <div class="controls">
        <div class="btn-row">
            <span onclick="skip(-5)">⏪ 5s</span>
            <span onclick="togglePlay()" id="play-ico" style="font-size: 30px;">▶</span>
            <span onclick="skip(5)">5s ⏩</span>
            <span id="like-btn" onclick="addLike()" style="cursor:pointer">🤍</span>
        </div>
        <div class="progress-bar" onclick="seek(event)">
            <div id="progress-fill"></div>
        </div>
    </div>
    <div class="vol-row">
        🔈 <input type="range" min="0" max="1" step="0.1" oninput="setVol(this.value)" style="accent-color: var(--accent);">
    </div>
</footer>

<audio id="player"></audio>

<script>
    let db = JSON.parse(localStorage.getItem('TTK_FINAL_DATA')) || [];
    let currentTrack = null;
    const player = document.getElementById('player');
    const likeTracker = JSON.parse(localStorage.getItem('TTK_LIKE_TRACKER')) || {};
    const viewTracker = JSON.parse(localStorage.getItem('TTK_VIEW_TRACKER')) || {};

    function showUpload() { document.getElementById('upload-modal').style.display = 'block'; }

    async function processUpload() {
        const title = document.getElementById('up-title').value;
        const artist = document.getElementById('up-artist').value;
        const audioFile = document.getElementById('up-audio').files[0];
        const imageFile = document.getElementById('up-image').files[0];

        if(!title || !audioFile || !imageFile) return alert("Fill everything out!");

        const audioBase64 = await toBase64(audioFile);
        const imageBase64 = await toBase64(imageFile);

        const track = {
            id: Date.now(),
            title, artist,
            audio: audioBase64,
            img: imageBase64,
            views: 0,
            likes: 0
        };

        db.push(track);
        render();
        document.getElementById('upload-modal').style.display = 'none';
    }

    const toBase64 = file => new Promise((res, rej) => {
        const reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => res(reader.result);
    });

    function render() {
        const grid = document.getElementById('grid');
        grid.innerHTML = db.map(s => `
            <div class="card" onclick="playSong(${s.id})">
                <img src="${s.img}">
                <div style="margin-top:10px"><b>${s.title}</b></div>
                <div style="color:#777; font-size:14px">${s.artist}</div>
                <div class="stats">
                    <span>👁 ${s.views}</span>
                    <span>❤️ ${s.likes}</span>
                </div>
            </div>
        `).join('');
    }

    function playSong(id) {
        const trackKey = `track_${id}`;
        const lastPlay = viewTracker[trackKey] || 0;
        if(Date.now() - lastPlay < 12 * 60 * 60 * 1000) return alert("You can't play this track more than 5 times an hour.");

        currentTrack = db.find(t => t.id === id);
        setTimeout(() => {
            currentTrack.views++;
            viewTracker[trackKey] = Date.now();
            localStorage.setItem('TTK_VIEW_TRACKER', JSON.stringify(viewTracker));
            render();
        }, 5000); // Count view after 5s play

        player.src = currentTrack.audio;
        player.play();
        document.getElementById('now-title').innerText = currentTrack.title;
        document.getElementById('now-artist').innerText = currentTrack.artist;
        document.getElementById('now-img').src = currentTrack.img;
        document.getElementById('play-ico').innerText = "⏸";
    }

    function togglePlay() {
        if(player.paused) { player.play(); document.getElementById('play-ico').innerText = "⏸"; }
        else { player.pause(); document.getElementById('play-ico').innerText = "▶"; }
    }

    function skip(sec) { player.currentTime += sec; }
    function setVol(v) { player.volume = v; }

    function addLike() {
        if(!currentTrack) return;
        const likeKey = `like_${currentTrack.id}`;
        if(likeTracker[likeKey]) return;
        currentTrack.likes++;
        likeTracker[likeKey] = true;
        localStorage.setItem('TTK_LIKE_TRACKER', JSON.stringify(likeTracker));
        render();
    }

    player.ontimeupdate = () => {
        const p = (player.currentTime / player.duration) * 100 || 0;
        document.getElementById('progress-fill').style.width = p + "%";
    };

    function seek(e) {
        const rect = e.target.getBoundingClientRect();
        player.currentTime = ((e.clientX - rect.left) / rect.width) * player.duration;
    }

    function saveAll() {
        localStorage.setItem('TTK_FINAL_DATA', JSON.stringify(db));
        alert("TTK SYSTEM SECURED: All songs, images, and stats saved!");
    }

    render();
</script>
</body>
</html>
