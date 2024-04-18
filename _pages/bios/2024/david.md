<div class="wrapper-snake" aria-label="A playable game of Snake">
  <div class="game-area">
    <canvas id="snake" width="600" height="360" tabindex="1"></canvas>
    <div class="overlay">
      <p class="game-msg"></p>
      <p id="finalScore"></p>
      <div class="buttons">
        <div id="startBtn" class="game-btn">Start Game</div>
        <div id="resumeBtn" class="game-btn hidden">Resume</div>
      </div>
      <div class="difficulty">
        <p>Game difficulty</p>
        <div>
          <div class="radio-field">
            <input type="radio" name="difficulty" id="radioEasy" value="easy">
            <label for="radioEasy">Easy</label>
          </div>
          <div class="radio-field">
            <input type="radio" name="difficulty" id="radioNormal" value="normal" checked>
            <label for="radioNormal">Normal</label>
          </div>
          <div class="radio-field">
            <input type="radio" name="difficulty" id="radioHard" value="hard">
            <label for="radioHard">Hard</label>
          </div>
        </div>
      </div>
    </div>
  </div>
  <div class="game-details">
    <input type="checkbox" name="sound" id="sound">
    <label for="sound">Sound on</label>
    <p class="hint"><span>Hint:</span> Use W, S, D, A or the arrows to move. Press spacebar to pause the game.</p>
    <p class="score-txt"><span>Score:</span> <span class="current-score">0</span></p>
  </div>
</div>
<!-- AUDIO FILES !-->
<audio id="eating" src="https://alexsalov.com/assets/snake/bite.mp3" preload="auto"></audio>
<audio id="hit_wall" src="https://alexsalov.com/assets/snake/hit_wall.mp3" preload="auto"></audio>
<audio id="pain" src="https://alexsalov.com/assets/snake/pain.mp3" preload="auto"></audio>
<audio id="snake_background" src="https://alexsalov.com/assets/snake/snake_background.mp3" preload="auto" loop></audio>

<script>
    (function() {
  'use strict';

  // User interface
  var pageCanvas = $('#snake'),
    startBtn = $('#startBtn'),
    resumeBtn = $('#resumeBtn'),
    gameMenu = $('.overlay'),
    gameOverMsg = $('.game-msg'),
    finalScore = $('#finalScore'),
    difficultyMenu = $('.difficulty'),
    scoreTxt = $('.current-score'),
    sound = $('#sound'),
    soundLabel = sound.nextSibling,
    bkgrMusic = $('#snake_background');

  bkgrMusic.volume = 0.25;

  // Game variables
  var canvasArea,
    w,
    h,
    snake,
    snakeLength = 10,
    food,
    score,
    difficulty = 'normal',
    gameLoop,
    dir, // Movement direction 
    over,
    hitType,
    speed, // Game speed
    objectSize = 12, // Proportional to canvas dimensions
    paused = false,
    muted = true, // Mute sound by default
    storage;

  // Game over messages
  var messages = {
    itself: [
      "Eat the food, not yourself, you stupid reptile!",
      "Good thing you can't poison yourself... I think...",
      "What a surprise, you bit yourself... AGAIN!",
      "T-t-t-tasty, tasty... You're so delicious ... to yourself at least.",
      "You took a solid bite out of your own ass. I hope it was worth it."
    ],
    wall: [
      "All in all, you're just another brick in the wall.",
      "That's a wall, bro - you can't go through it.",
      "You like hurting yourself, don't you?",
      "Hit the wall snake and don't you come back no more, no more, no more, no more!",
      "You crushed your pathetic snake skull once more."
    ]
  };

  /***** EVENT LISTENERS *****/
  addListener(window, 'load', function() {
    checkSound();

    storage = new Storage('snake_');

    // Check if a previous game has been paused
    if (storage.exists('paused') && storage.get('paused') === 'true') {
      paused = true;
      resumeBtn.classList.remove('hidden');
      difficultyMenu.classList.add('hidden');

      startBtn.innerHTML = 'Restart Game';
      scoreTxt.innerHTML = storage.get('score');
      difficulty = storage.get('difficulty');

      var radio = difficultyMenu.getElementsByTagName('input');
      var radioNum = radio.length;

      for (var i = 0; i < radioNum; i++) {
        if (radio[i].value === difficulty) radio[i].checked = true;
      }
    }

    // Instantiate canvas element
    canvasArea = new Canvas(pageCanvas, pageCanvas.width, pageCanvas.height, '#fff');
    w = canvasArea.width;
    h = canvasArea.height;
  });

  // Toggle sound
  addListener(sound, 'change', function() {
    checkSound();
  });

  addListener(startBtn, 'click', function() {
    if (!muted) bkgrMusic.play();

    initGame();
  });

  addListener(resumeBtn, 'click', function() {
    if (!over) resumeGame();
  });

  // Detect difficulty change
  addListener(difficultyMenu, 'change', function(e) {
    var elem = e.target;
    elem.checked = true;
    difficulty = elem.value;
  });

  // Detect space key press
  addListener(pageCanvas, 'keydown', function(e) {
    if (e.keyCode === 32 && !over) {
      (!paused) ? pauseGame(): resumeGame();
    }
  });

  /***** FUNCTIONS *****/

  /* UTILITY FUNCTIONS */

  // Get element
  function $(selector) {
    return document.querySelector(selector);
  }

  function addListener(elem, handler, callback) {
    elem.addEventListener(handler, callback);
  }

  /* OBJECT DECLARATION FUNCTIONS */

  // Canvas area
  function Canvas(canvas, width, height, color) {
    this.canvas = canvas;
    this.width = width;
    this.height = height;
    this.color = color;
    this.context = this.canvas.getContext('2d');
  }

  Canvas.prototype.rebuild = function() {
    this.context.fillStyle = this.color;
    this.context.fillRect(0, 0, this.width, this.height);
  };

  // Canvas object super class
  function CanvasObj(canvas, x, y, size, color) {
    this.context = canvas.context;
    this.x = x;
    this.y = y;
    this.size = size;
    this.color = color;
  }

  CanvasObj.prototype.draw = function() {
    this.context.fillStyle = this.color;
    this.context.fillRect(this.x * this.size, this.y * this.size, this.size, this.size);
  };

  // Snake object
  function Snake(canvas, x, y, size, color) {
    CanvasObj.call(this, canvas, x, y, size, color);

    this.length = snakeLength;
    this.pos = [];
  }

  // Inherit form CanvasObj class
  Snake.prototype = Object.create(CanvasObj.prototype);
  Snake.prototype.constructor = Snake;

  // Initialize the snake
  Snake.prototype.init = function() {
    for (var i = this.length - 1; i >= 0; i--) {
      this.pos.push({
        x: i + (Math.round(this.x / this.size)),
        y: Math.round(this.y / this.size)
      });
    }
  };

  Snake.prototype.draw = function() {
    var len = this.length;

    for (var i = 0; i < len; i++) {
      var square = this.pos[i];
      this.context.fillStyle = this.color;
      this.context.fillRect(square.x * this.size, square.y * this.size, this.size, this.size);
    }
  };

  // Update the position of the snake
  Snake.prototype.update = function() {
    var headX = this.pos[0].x;
    var headY = this.pos[0].y;

    // Get the directions
    addListener(document, 'keydown', function(e) {
      e.preventDefault(); // Prevent scroll when using arrow keys

      var key = e.keyCode;

      // Allow snake to be moved with W, S, D, A as well as the arrow keys
      if ((key === 37 || key === 65) && dir !== 'r') dir = 'l';
      else if ((key === 38 || key === 87) && dir !== 'd') dir = 'u';
      else if ((key === 39 || key === 68) && dir !== 'l') dir = 'r';
      else if ((key === 40 || key === 83) && dir !== 'u') dir = 'd';
    });

    // Directions
    switch (dir) {
      case 'l':
        headX--;
        break;
      case 'u':
        headY--;
        break;
      case 'r':
        headX++;
        break;
      case 'd':
        headY++;
        break;
    }

    // Move snake
    var tail = this.pos.pop();
    tail.x = headX;
    tail.y = headY;
    this.pos.unshift(tail);

    // Wall Collision
    if (headX >= w / this.size || headX <= -1 || headY >= h / this.size || headY <= -1) {
      if (!over) {
        hitType = 'wall';
        over = true;
        gameOver();
      }
    }

    // Food collision
    if (headX === food.x && headY === food.y) {
      if (!muted) $('#eating').play();

      food = new CanvasObj(canvasArea, getFoodX(), getFoodY(), objectSize, '#ff0000');
      tail = {
        x: headX,
        y: headY
      };
      this.pos.unshift(tail);
      this.length++;
      score += getValue('score', difficulty);
      scoreTxt.innerHTML = score;

      // Increase game speed
      if (speed <= 45) speed += getValue('speed', difficulty);
      clearInterval(gameLoop);
      gameLoop = setInterval(drawCanvas, 1000 / speed);
    } else {
      // Check collision between snake parts
      var len = this.pos.length;

      for (var i = 1; i < len; i++) {
        var square = this.pos[i];

        if ((headX === square.x && headY === square.y) && !over) {
          hitType = 'itself';
          over = true;
          gameOver();
        }
      }
    }
  };

  // Helper class for working with localStorage
  function Storage(prefix) {
    this.prefix = prefix; // Namespacing the values saved to storage
  }

  Storage.prototype.get = function(key) {
    return localStorage.getItem(this.prefix + key);
  };

  Storage.prototype.set = function(key, value) {
    localStorage.setItem(this.prefix + key, value);
  };

  Storage.prototype.getAsNum = function(key) {
    return Number(this.get(key));
  };

  Storage.prototype.exists = function(key) {
    return this.get(key) !== null;
  };

  Storage.prototype.length = function() {
    return localStorage.length;
  };

  Storage.prototype.key = function(index) {
    return localStorage.key(index);
  };

  Storage.prototype.remove = function(key) {
    localStorage.removeItem(this.prefix + key);
  };

  // Remove all game state data
  Storage.prototype.removeAll = function() {
    // Run loop backwards since items are being deleted
    for (var i = this.length() - 1; i >= 0; i--) {
      var key = this.key(i);

      if (key.indexOf(this.prefix) > -1) localStorage.removeItem(key);
    }
  };

  Storage.prototype.clear = function() {
    localStorage.clear();
  };

  /* GAME FUNCTIONS */
  function drawCanvas() {
    canvasArea.rebuild();
    snake.draw();
    snake.update();
    food.draw();
  }

  function initGame() {
    gameMenu.classList.add('hidden');
    difficultyMenu.classList.add('hidden');

    storage.removeAll();

    snake = new Snake(canvasArea, getSnakeX(), getSnakeY(), objectSize, '#000');
    snake.init();
    food = new CanvasObj(canvasArea, getFoodX(), getFoodY(), objectSize, '#ff0000');
    dir = 'r';
    over = false;
    score = 0;
    speed = 10;
    paused = false;

    if (gameLoop !== undefined) clearInterval(gameLoop);
    gameLoop = setInterval(drawCanvas, 1000 / speed);

    scoreTxt.innerHTML = score;
    startBtn.innerHTML = 'Restart Game';
    gameOverMsg.innerHTML = '';
    finalScore.innerHTML = '';
  }

  function gameOver() {
    clearInterval(gameLoop);

    if (!muted) {
      bkgrMusic.pause();

      if (hitType === 'wall') {
        $('#hit_wall').play();
      } else if (hitType === 'itself') {
        $('#pain').play();
      }
    }

    gameMenu.classList.remove('hidden');
    difficultyMenu.classList.remove('hidden');
    resumeBtn.classList.add('hidden');

    // Show end message
    gameOverMsg.innerHTML = messages[hitType][Math.floor(Math.random() * messages[hitType].length)];
    finalScore.innerHTML = 'Final score: ' + score;
  }

  function pauseGame() {
    if (!muted) bkgrMusic.pause();

    gameMenu.classList.remove('hidden');
    resumeBtn.classList.remove('hidden');

    paused = true;

    // Save game state to localStorage
    storage.set('score', score);
    storage.set('speed', speed);
    storage.set('direction', dir);
    storage.set('difficulty', difficulty);
    storage.set('length', snake.length);
    storage.set('foodX', food.x);
    storage.set('foodY', food.y);
    storage.set('paused', 'true');

    for (var i = 0; i < snake.length; i++) {
      storage.set('posX' + i, snake.pos[i].x);
      storage.set('posY' + i, snake.pos[i].y);
    }

    clearInterval(gameLoop);
  }

  // Continue the game from the point where it was last paused
  function resumeGame() {

    if (!muted) bkgrMusic.play();

    gameMenu.classList.add('hidden');
    paused = false;

    // Restore game state
    score = storage.getAsNum('score');
    speed = storage.getAsNum('speed');
    dir = storage.get('direction');
    difficulty = storage.get('difficulty');

    snake = new Snake(canvasArea, getSnakeX(), getSnakeY(), objectSize, '#000');
    snake.length = storage.getAsNum('length');
    snake.pos = [];

    for (var i = 0; i < snake.length; i++) {
      snake.pos.push({
        x: storage.getAsNum('posX' + i),
        y: storage.getAsNum('posY' + i)
      });
    }

    food = new CanvasObj(canvasArea, storage.getAsNum('foodX'), storage.getAsNum('foodY'), objectSize, '#ff0000');
    gameLoop = setInterval(drawCanvas, 1000 / speed);

    storage.removeAll();
  }

  // Generate random coordinates for canvas objects
  function getSnakeX() {
    return getRandomNum(w - (snakeLength * objectSize + 100)); // Place snake at least 100px from canvas edge
  }

  function getSnakeY() {
    return getRandomNum(h - objectSize);
  }

  function getFoodX() {
    return getRandomNum((w - objectSize) / objectSize);
  }

  function getFoodY() {
    return getRandomNum((h - objectSize) / objectSize);
  }

  function getRandomNum(max) {
    return Math.round(Math.random() * max);
  }

  // Get property value according to difficulty level
  function getValue(property, difficulty) {
    switch (difficulty) {
      case 'easy':
        return (property === 'speed') ? 0.25 : 5;
      case 'normal':
        return (property === 'speed') ? 0.5 : 10;
      case 'hard':
        return (property === 'speed') ? 1 : 20;
    }
  }

  // Check if sound is enabled
  function checkSound() {
    if (sound.checked && paused === false && gameLoop !== undefined) {
      muted = false;
      soundLabel.innerHTML = 'Sound off';
      bkgrMusic.play();
    } else {
      sound.checked = false;
      muted = true;
      soundLabel.innerHTML = 'Sound on';
      bkgrMusic.pause();
    }
  }
}());
</script>