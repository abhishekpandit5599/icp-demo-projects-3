# icp-demo-projects-3

## SNAKE GAME

`main.mo`

```js
actor {
   stable var highestScore : Nat = 0;

  public func updateHighestScore(score : Nat) {
    if (score > highestScore) {
      highestScore := score;
    };
  };

  public query func highestScoreHandler() : async Nat {
    return highestScore;
  }
};

```

`main.css`

```js
body {
  text-align: center;
  font-family: "Arial", sans-serif;
  background-color: #f5f5f5;
  margin: 0;
  padding: 0;
}

h1 {
  color: #007bff;
  font-size: 36px;
  font-weight: bold;
  margin-top: 20px;
  font-family: "Lucida Sans", "Lucida Sans Regular", "Lucida Grande",
    "Lucida Sans Unicode", Geneva, Verdana, sans-serif;
}

canvas {
  border: 2px solid #555;
  background-color: #eee;
  margin: 20px auto;
  display: block;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.alert {
  padding: 10px;
  margin: 10px;
  color: #fff;
  font-weight: bold;
  border-radius: 5px;
  display: inline-block;
}

.alert-success {
  background-color: #28a745;
}

.alert-danger {
  background-color: #dc3545;
}

.button {
  background-color: #007bff;
  color: #fff;
  border: none;
  padding: 10px 15px;
  font-size: 16px;
  cursor: pointer;
  border-radius: 5px;
  margin-top: 20px;
}

.button:hover {
  background-color: #0056b3;
}

```

`index.html`

```js
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width" />
    <title>towerBuilding</title>
    <base href="/" />
    <link rel="icon" href="favicon.ico" />
    <link type="text/css" rel="stylesheet" href="main.css" />
  </head>
  <body>
    <h1>Snake Game 🪱</h1>
    <canvas id="stage" height="400" width="520"></canvas>
  </body>
</html>

```

`index.js`

```js
import { towerBuilding_backend } from "../../declarations/towerBuilding_backend";

var Game = Game || {};
var Keyboard = Keyboard || {};
var Component = Component || {};

/**
 * Keyboard Map
 */
Keyboard.Keymap = {
  37: "left",
  38: "up",
  39: "right",
  40: "down",
};

/**
 * Keyboard Events
 */
Keyboard.ControllerEvents = function () {
  var self = this;
  this.pressKey = null;
  this.keymap = Keyboard.Keymap;

  document.onkeydown = function (event) {
    self.pressKey = event.which;
  };

  this.getKey = function () {
    return this.keymap[this.pressKey];
  };
};

/**
 * Game Component Stage
 */
Component.Stage = function (canvas, conf, callback) {
  this.keyEvent = new Keyboard.ControllerEvents();
  this.width = canvas.width;
  this.height = canvas.height;
  this.length = [];
  this.food = {};
  this.direction = "right";
  this.isGameOver = false;
  this.conf = {
    cw: 10,
    size: 5,
    fps: 1000,
  };

  if (typeof conf === "object") {
    for (var key in conf) {
      if (conf.hasOwnProperty(key)) {
        this.conf[key] = conf[key];
      }
    }
  }

  // Fetch the highest score from the backend
  towerBuilding_backend
    .highestScoreHandler()
    .then((highestScore) => {
      this.highestScore = highestScore;
      this.score = 0;
      if (typeof callback === "function") {
        callback(this);
      }
    })
    .catch((error) => {
      console.error("Error fetching highest score:", error);
      this.highestScore = 0;
    });
};

/**
 * Game Component Snake
 */
Component.Snake = function (canvas, conf, callback) {
  new Component.Stage(canvas, conf, (stage) => {
    this.stage = stage;

    this.initSnake = function () {
      this.stage.length = [];
      for (var i = 0; i < this.stage.conf.size; i++) {
        this.stage.length.push({ x: i, y: 0 });
      }
    };

    this.initSnake();

    this.initFood = function () {
      this.stage.food = {
        x: Math.round(
          (Math.random() * (this.stage.width - this.stage.conf.cw)) /
            this.stage.conf.cw
        ),
        y: Math.round(
          (Math.random() * (this.stage.height - this.stage.conf.cw)) /
            this.stage.conf.cw
        ),
      };
    };

    this.initFood();

    this.restart = async function () {
      this.stage.length = [];
      this.stage.food = {};
      this.stage.score = 0;
      this.stage.direction = "right";
      this.stage.keyEvent.pressKey = null;
      this.stage.isGameOver = false;
      this.initSnake();
      this.initFood();
    };

    if (typeof callback === "function") {
      callback(this);
    }
  });
};

/**
 * Game Draw
 */
Game.Draw = function (context, snake) {
  this.drawStage = async function () {
    if (snake.stage.length.length === 0) {
      console.error("Snake length array is empty.");
      return;
    }

    var keyPress = snake.stage.keyEvent.getKey();
    if (typeof keyPress != "undefined") {
      snake.stage.direction = keyPress;
    }

    context.fillStyle = "white";
    context.fillRect(0, 0, snake.stage.width, snake.stage.height);

    var nx = snake.stage.length[0].x;
    var ny = snake.stage.length[0].y;

    switch (snake.stage.direction) {
      case "right":
        nx++;
        break;
      case "left":
        nx--;
        break;
      case "up":
        ny--;
        break;
      case "down":
        ny++;
        break;
    }

    if (!snake.stage.isGameOver && this.collision(nx, ny)) {
      snake.stage.isGameOver = true;
      await towerBuilding_backend.updateHighestScore(snake.stage.score);

      // Fetch and update the highest score after the game ends
      try {
        const newHighestScore =
          await towerBuilding_backend.highestScoreHandler();
        snake.stage.highestScore = newHighestScore;
      } catch (error) {
        console.error("Error fetching updated highest score:", error);
      }

      const restartGame = confirm(
        "Game over! Your score: " +
          snake.stage.score +
          ". Do you want to restart?"
      );
      if (restartGame) {
        await snake.restart();
      } else {
      }
      return;
    }

    if (nx === snake.stage.food.x && ny === snake.stage.food.y) {
      var tail = { x: nx, y: ny };
      snake.stage.score++;
      snake.initFood();
    } else {
      var tail = snake.stage.length.pop();
      tail.x = nx;
      tail.y = ny;
    }
    snake.stage.length.unshift(tail);

    for (var i = 0; i < snake.stage.length.length; i++) {
      var cell = snake.stage.length[i];
      this.drawCell(cell.x, cell.y);
    }

    this.drawCell(snake.stage.food.x, snake.stage.food.y);
    context.fillText("Highest Score: " + snake.stage.highestScore, 5, 20);
    context.fillText(
      "Current Score: " + snake.stage.score,
      5,
      snake.stage.height - 5
    );
  };

  this.drawCell = function (x, y) {
    context.fillStyle = "rgb(170, 170, 170)";
    context.beginPath();
    context.arc(
      x * snake.stage.conf.cw + 6,
      y * snake.stage.conf.cw + 6,
      4,
      0,
      2 * Math.PI,
      false
    );
    context.fill();
  };

  this.collision = function (nx, ny) {
    if (
      nx == -1 ||
      nx == snake.stage.width / snake.stage.conf.cw ||
      ny == -1 ||
      ny == snake.stage.height / snake.stage.conf.cw
    ) {
      return true;
    }
    return false;
  };
};

/**
 * Game Snake
 */
Game.Snake = function (elementId, conf) {
  var canvas = document.getElementById(elementId);
  var context = canvas.getContext("2d");

  new Component.Snake(canvas, conf, (snake) => {
    var gameDraw = new Game.Draw(context, snake);
    setInterval(function () {
      gameDraw.drawStage();
    }, snake.stage.conf.fps);
  });
};

/**
 * Window Load
 */
window.onload = function () {
  const startGame = confirm("Do you want to start the game?");
  if (startGame) {
    new Game.Snake("stage", { fps: 120, size: 4 });
    window.addEventListener("beforeunload", function (event) {
      alert("Game over! Thanks for playing!");
    });
  } else {
    alert("Game not started. Have a great day!");
  }
};

```