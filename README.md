<!DOCTYPE html>
<html lang="cs">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Převody jednotek – Hra</title>
<style>
body {
  font-family: 'Comic Sans MS', sans-serif;
  background: linear-gradient(to right, #a1c4fd, #c2e9fb);
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  overflow: hidden;
}
.container {
  background-color: #ffffffcc;
  padding: 30px;
  border-radius: 20px;
  box-shadow: 0 5px 20px rgba(0,0,0,0.3);
  text-align: center;
  width: 480px;
  position: relative;
}
h2 { margin-bottom: 10px; }
select, input {
  padding: 8px;
  margin: 5px;
  border-radius: 8px;
  border: 2px solid #ccc;
}
input { width: 120px; }
button {
  padding: 8px 20px;
  margin: 5px;
  border: none;
  background-color: #007bff;
  color: white;
  border-radius: 10px;
  cursor: pointer;
  font-size: 1rem;
}
button:hover { background-color: #0056b3; }
.feedback { margin-top: 15px; font-weight: bold; font-size: 1.1rem; min-height: 30px; white-space: pre-line; }
.score { position: absolute; top: 10px; right: 20px; font-weight: bold; font-size: 1.1rem; }
.fade-in { animation: fadeIn 0.5s ease-in-out; }
@keyframes fadeIn {
  from { opacity: 0; transform: scale(0.95); }
  to { opacity: 1; transform: scale(1); }
}
.confetti { position: fixed; width: 10px; height: 10px; top: -10px; z-index: 1000; pointer-events: none; opacity: 0.9; animation: fall 3s linear forwards; }
@keyframes fall { to { transform: translateY(100vh) rotate(360deg); opacity: 0; } }
</style>
</head>
<body>
<div class="container">
  <div class="score">Otázka: <span id="current">0</span>/<span id="total">0</span> | Skóre: <span id="score">0</span></div>
  <h2>Převody jednotek</h2>

  <!-- výběr úloh – jen na začátku nebo po skončení -->
  <div id="setup">
    <label>Vyber typ úloh:</label><br>
    <select id="category">
      <option value="all">Všechny</option>
      <option value="delka">Délka</option>
      <option value="obsah">Obsah</option>
      <option value="objem">Objem</option>
    </select>
    <button onclick="startGame()">Start</button>
  </div>

  <p id="question"></p>
  <input type="number" id="answer" placeholder="Zadej odpověď" style="display:none;">
  <br>
  <button onclick="checkAnswer()" id="checkBtn" style="display:none;">Ověřit</button>
  <button onclick="newQuestion()" id="nextBtn" style="display:none;">Nová otázka</button>
  <p class="feedback" id="feedback"></p>
</div>

<script>
let score = 0;
let questionNumber = 0;
let correctAnswers = 0;
let questions = [];
let currentQuestion = {};

const delka = [
  { value: 3, from: 'm', to: 'cm', factor: 100 },
  { value: 30, from: 'dm', to: 'mm', factor: 100 },
  { value: 0.3, from: 'km', to: 'm', factor: 1000 },
  { value: 3.5, from: 'km', to: 'dm', factor: 10000 },
  { value: 350, from: 'cm', to: 'm', factor: 0.01 },
  { value: 4000, from: 'mm', to: 'dm', factor: 0.01 },
  { value: 4500, from: 'dm', to: 'km', factor: 0.0001 },
  { value: 850, from: 'm', to: 'km', factor: 0.001 },
  { value: 0.245, from: 'm', to: 'mm', factor: 1000 },
  { value: 10.35, from: 'km', to: 'cm', factor: 100000 }
];

const obsah = [
  { value: 25, from: 'cm<sup>2</sup>', to: 'dm<sup>2</sup>', factor: 0.01 },
  { value: 250, from: 'dm<sup>2</sup>', to: 'cm<sup>2</sup>', factor: 100 },
  { value: 4000, from: 'm<sup>2</sup>', to: 'dm<sup>2</sup>', factor: 100 },
  { value: 550, from: 'dm<sup>2</sup>', to: 'mm<sup>2</sup>', factor: 10000 },
  { value: 3, from: 'mm<sup>2</sup>', to: 'dm<sup>2</sup>', factor: 0.0001 },
  { value: 50, from: 'm<sup>2</sup>', to: 'a', factor: 0.01 },
  { value: 280, from: 'a', to: 'ha', factor: 0.01 },
  { value: 3420, from: 'ha', to: 'km<sup>2</sup>', factor: 0.01 },
  { value: 5, from: 'km<sup>2</sup>', to: 'm<sup>2</sup>', factor: 1000000 },
  { value: 6000, from: 'dm<sup>2</sup>', to: 'a', factor: 0.01 }
];

const objem = [
  { value: 15, from: 'm<sup>3</sup>', to: 'cm<sup>3</sup>', factor: 1000000 },
  { value: 800, from: 'dm<sup>3</sup>', to: 'm<sup>3</sup>', factor: 0.001 },
  { value: 25, from: 'mm<sup>3</sup>', to: 'cm<sup>3</sup>', factor: 0.001 },
  { value: 7500, from: 'dm<sup>3</sup>', to: 'cm<sup>3</sup>', factor: 1000 },
  { value: 4, from: 'km<sup>3</sup>', to: 'm<sup>3</sup>', factor: 1000000000 },
  { value: 2, from: 'l', to: 'dm<sup>3</sup>', factor: 1 },
  { value: 20, from: 'l', to: 'hl', factor: 0.01 },
  { value: 750, from: 'm<sup>3</sup>', to: 'l', factor: 1000 },
  { value: 50, from: 'ml', to: 'l', factor: 0.001 },
  { value: 480, from: 'ml', to: 'm<sup>3</sup>', factor: 0.000001 }
];

function startGame() {
  // skryjeme setup
  document.getElementById('setup').style.display = 'none';

  const choice = document.getElementById('category').value;
  if (choice === 'delka') questions = [...delka];
  else if (choice === 'obsah') questions = [...obsah];
  else if (choice === 'objem') questions = [...objem];
  else questions = [...delka, ...obsah, ...objem];

  score = 0;
  questionNumber = 0;
  correctAnswers = 0;

  document.getElementById('total').innerText = questions.length;
  document.getElementById('score').innerText = 0;
  document.getElementById('current').innerText = 0;

  document.getElementById('answer').style.display = 'inline-block';
  document.getElementById('checkBtn').style.display = 'inline-block';
  document.getElementById('nextBtn').style.display = 'inline-block';

  newQuestion();
}

function newQuestion() {
  if (questionNumber >= questions.length) {
    endGame();
    return;
  }
  currentQuestion = questions[questionNumber];
  const qEl = document.getElementById('question');
  qEl.innerHTML = `${currentQuestion.value} ${currentQuestion.from} = ? ${currentQuestion.to}`;
  qEl.classList.remove('fade-in');
  void qEl.offsetWidth;
  qEl.classList.add('fade-in');

  const input = document.getElementById('answer');
  input.value = '';
  input.style.borderColor = '#ccc';
  input.style.backgroundColor = 'white';

  document.getElementById('feedback').innerText = '';
  document.getElementById('current').innerText = questionNumber + 1;
}

function checkAnswer() {
  const input = document.getElementById('answer');
  const userAnswer = parseFloat(input.value);
  const correctAnswer = currentQuestion.value * currentQuestion.factor;
  const feedback = document.getElementById('feedback');

  if (userAnswer === correctAnswer) {
    feedback.style.color = 'green';
    feedback.innerText = 'Správně! 🎉';
    score++;
    correctAnswers++;
    input.style.borderColor = 'green';
    input.style.backgroundColor = '#d4edda';
  } else {
    feedback.style.color = 'red';
    feedback.innerText = `Špatně 😢 Správná odpověď je ${correctAnswer}`;
    input.style.borderColor = 'red';
    input.style.backgroundColor = '#f8d7da';
  }
  document.getElementById('score').innerText = score;
  questionNumber++;
  setTimeout(newQuestion, 1000);
}

function endGame() {
  document.getElementById('question').innerText = 'Hra dokončena!';
  document.getElementById('answer').style.display = 'none';
  document.getElementById('checkBtn').style.display = 'none';
  document.getElementById('nextBtn').style.display = 'none';

  const feedback = document.getElementById('feedback');
  let hodnoceni = '';
  if (correctAnswers === questions.length) {
    hodnoceni = "Perfektní! 🎉 Máš 100 %!";
    launchConfetti(100);
  } else if (correctAnswers >= 15) {
    hodnoceni = "Skvělý výsledek! 🌟";
  } else if (correctAnswers >= 11) {
    hodnoceni = "Dobrá práce, ale můžeš to mít ještě lepší 👍";
  } else {
    hodnoceni = "Zkus to ještě jednou 💪";
  }

  feedback.innerText =
    `Zodpovězeno správně ${correctAnswers}/${questions.length} otázek.\n${hodnoceni}\n\nNejdou ti převody jednotek nebo chceš zlepšit své dovednosti – napiš mi na: pavla.drex@seznam.cz`;

  // tlačítko "Zkusit znovu"
  const retryBtn = document.createElement('button');
  retryBtn.innerText = "Zkusit znovu";
  retryBtn.onclick = () => {
    document.getElementById('feedback').innerText = "";
    document.getElementById('question').innerText = "";
    retryBtn.remove();
    // znovu zobrazíme setup
    document.getElementById('setup').style.display = 'block';
  };
  document.querySelector('.container').appendChild(retryBtn);
}

// Konfety
function launchConfetti(count) {
  for (let i = 0; i < count; i++) {
    const confetti = document.createElement('div');
    confetti.classList.add('confetti');
    confetti.style.left = Math.random() * window.innerWidth + 'px';
    confetti.style.backgroundColor = `hsl(${Math.random() * 360},70%,50%)`;
    confetti.style.width = confetti.style.height = (Math.random() * 8 + 5) + 'px';
    confetti.style.animationDuration = (Math.random() * 2 + 2) + 's';
    document.body.appendChild(confetti);
    setTimeout(() => confetti.remove(), 3000);
  }
}
</script>
</body>
</html>
