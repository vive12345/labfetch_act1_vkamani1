<template>
  <div id="app">
    <h1>Jeopardy - Enhanced Edition</h1>

    <!-- Game Configuration (before game starts) -->
    <div v-if="!gameStarted && !loading" class="game-config">
      <div class="config-section">
        <label for="numPlayers">Number of players:</label>
        <select id="numPlayers" v-model="numPlayers" class="config-input">
          <option v-for="n in 5" :key="n" :value="n + 1">{{ n + 1 }}</option>
        </select>
      </div>

      <div class="config-section">
        <label for="numCategories">Number of categories:</label>
        <select id="numCategories" v-model="numCategories" class="config-input">
          <option v-for="n in 5" :key="n" :value="n + 1">{{ n + 1 }}</option>
        </select>
      </div>

      <button @click="initializeGame" class="start-btn">Start Game</button>
    </div>

    <!-- Game Board -->
    <div class="game-board" v-if="gameStarted">
      <table class="board-table">
        <!-- Category Headers -->
        <thead>
          <tr>
            <th
              v-for="category in selectedCategories"
              :key="category.id"
              class="category-header"
            >
              {{ category.name }}
            </th>
          </tr>
        </thead>
        <!-- Question Grid -->
        <tbody>
          <tr v-for="row in 5" :key="row" class="question-row">
            <td
              v-for="(category, colIndex) in selectedCategories"
              :key="colIndex"
              class="question-cell"
            >
              <button
                v-if="!gameBoard[colIndex][row - 1].answered"
                @click="selectQuestion(colIndex, row - 1)"
                class="question-button"
              >
                ${{ gameBoard[colIndex][row - 1].value }}
              </button>
              <div
                v-else
                class="answered-cell"
                :class="{
                  correct: gameBoard[colIndex][row - 1].correct,
                  incorrect: !gameBoard[colIndex][row - 1].correct,
                }"
              >
                P{{ gameBoard[colIndex][row - 1].playerId }}
              </div>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Loading message -->
    <div v-if="loading" class="loading">Loading questions... Please wait.</div>

    <!-- Double Jeopardy Wager Input -->
    <div v-if="showDoubleJeopardyWager" class="double-jeopardy-wager">
      <h3>🎯 Double Jeopardy!</h3>
      <p>Player {{ currentPlayer }}, you can wager up to ${{ maxWager }}.</p>
      <div class="wager-input">
        <label>Enter your wager: $</label>
        <input
          type="number"
          v-model.number="wagerAmount"
          :max="maxWager"
          :min="minWager"
          class="wager-field"
        />
        <button
          @click="submitWager"
          :disabled="!isValidWager"
          class="wager-btn"
        >
          Submit Wager
        </button>
      </div>
      <p class="wager-info">
        Minimum wager: ${{ minWager }}, Maximum wager: ${{ maxWager }}
      </p>
    </div>

    <!-- Question Display Area -->
    <div
      v-if="currentQuestion && !showDoubleJeopardyWager"
      class="question-area"
    >
      <div v-if="isDoubleJeopardy" class="double-jeopardy-indicator">
        🎯 DOUBLE JEOPARDY - Wagering ${{ currentWager }}
        <div v-if="autoWager" class="auto-wager-message">
          (Auto-wager: Your balance was less than face value)
        </div>
      </div>
      <p class="question-header">
        Player {{ currentPlayer }} selects
        {{ selectedCategories[currentQuestion.categoryIndex].name }} for ${{
          isDoubleJeopardy ? currentWager : currentQuestion.value
        }}:
      </p>
      <p class="question-text">{{ currentQuestion.question }}</p>
      <div class="answer-buttons">
        <label>
          <input type="radio" v-model="playerAnswer" value="True" /> True
        </label>
        <label>
          <input type="radio" v-model="playerAnswer" value="False" /> False
        </label>
        <button
          @click="submitAnswer"
          :disabled="!playerAnswer"
          class="submit-btn"
        >
          Submit
        </button>
      </div>
    </div>

    <!-- Result Display -->
    <div v-if="showResult" class="result-area">
      <p class="result-message">{{ resultMessage }}</p>
      <p class="next-player">{{ nextPlayerMessage }}</p>
    </div>

    <!-- Game Over -->
    <div v-if="gameOver" class="game-over">
      <h2>{{ winnerMessage }}</h2>
    </div>

    <!-- Player Scores -->
    <div class="scores" v-if="gameStarted">
      <h3>Scores:</h3>
      <div class="scores-horizontal">
        <div v-for="(score, index) in playerScores" :key="index" class="score">
          Player {{ index + 1 }}: ${{ score }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "JeopardyGameEnhanced",
  data() {
    return {
      // Game configuration
      numPlayers: 3,
      numCategories: 4,

      // Game state
      gameStarted: false,
      loading: false,
      gameOver: false,

      // API data
      allCategories: [],
      selectedCategories: [],
      excludedCategories: [10, 13, 21, 26, 27, 29, 30, 32],

      // Game board
      gameBoard: [],
      questionsAnswered: 0,
      totalQuestions: 0,

      // Current game state
      currentPlayer: 1,
      playerScores: [],
      currentQuestion: null,
      playerAnswer: "",
      showResult: false,
      resultMessage: "",
      nextPlayerMessage: "",

      // Double Jeopardy
      isDoubleJeopardy: false,
      showDoubleJeopardyWager: false,
      wagerAmount: 0,
      currentWager: 0,
      maxWager: 0,
      minWager: 0,
      doubleJeopardyCount: 0,
      autoWager: false,

      // API timing
      lastApiCall: 0,
      apiDelay: 5000,
    };
  },

  computed: {
    winnerMessage() {
      if (!this.gameOver) return "";
      const maxScore = Math.max(...this.playerScores);
      const winnerId =
        this.playerScores.findIndex((score) => score === maxScore) + 1;
      return `Player ${winnerId} has won the game!`;
    },

    isValidWager() {
      return (
        this.wagerAmount >= this.minWager && this.wagerAmount <= this.maxWager
      );
    },
  },

  methods: {
    async initializeGame() {
      this.loading = true;
      try {
        // Initialize player scores based on selected number of players
        this.playerScores = new Array(this.numPlayers).fill(0);
        this.totalQuestions = this.numCategories * 5;
        this.doubleJeopardyCount = 0;

        await this.fetchCategories();
        await this.setupGameBoard();
        this.gameStarted = true;
      } catch (error) {
        console.error("Error initializing game:", error);
        alert("Error loading game. Please try again.");
      } finally {
        this.loading = false;
      }
    },

    async fetchCategories() {
      try {
        const response = await fetch("https://opentdb.com/api_category.php");
        const data = await response.json();

        // Filter out excluded categories
        this.allCategories = data.trivia_categories.filter(
          (cat) => !this.excludedCategories.includes(cat.id)
        );

        // Randomly select the specified number of categories
        this.selectedCategories = this.getRandomCategories(this.numCategories);
        this.lastApiCall = Date.now();
      } catch (error) {
        throw new Error("Failed to fetch categories");
      }
    },

    getRandomCategories(count) {
      const shuffled = [...this.allCategories].sort(() => 0.5 - Math.random());
      return shuffled.slice(0, count);
    },

    async setupGameBoard() {
      this.gameBoard = [];

      for (
        let categoryIndex = 0;
        categoryIndex < this.numCategories;
        categoryIndex++
      ) {
        const categoryColumn = [];
        const category = this.selectedCategories[categoryIndex];

        for (let questionIndex = 0; questionIndex < 5; questionIndex++) {
          const value = (questionIndex + 1) * 100;
          let difficulty;

          if (value <= 200) difficulty = "easy";
          else if (value <= 400) difficulty = "medium";
          else difficulty = "hard";

          categoryColumn.push({
            value: value,
            difficulty: difficulty,
            categoryId: category.id,
            categoryIndex: categoryIndex,
            questionIndex: questionIndex,
            answered: false,
            correct: false,
            playerId: null,
            question: null,
            correctAnswer: null,
          });
        }

        this.gameBoard.push(categoryColumn);
      }
    },

    checkForDoubleJeopardy() {
      // 10% chance for Double Jeopardy
      const random = Math.random();
      return random < 0.1;
    },

    async selectQuestion(categoryIndex, questionIndex) {
      if (
        this.currentQuestion ||
        this.showResult ||
        this.showDoubleJeopardyWager
      )
        return;

      const questionData = this.gameBoard[categoryIndex][questionIndex];
      if (questionData.answered) return;

      // Check for Double Jeopardy
      this.isDoubleJeopardy = this.checkForDoubleJeopardy();

      if (this.isDoubleJeopardy) {
        this.doubleJeopardyCount++;
        this.setupDoubleJeopardyWager(questionData);
        return;
      }

      // Regular question flow
      await this.fetchAndDisplayQuestion(questionData);
    },

    setupDoubleJeopardyWager(questionData) {
      const playerBalance = this.playerScores[this.currentPlayer - 1];
      const faceValue = questionData.value;

      this.maxWager = Math.max(playerBalance, faceValue);
      this.minWager = Math.min(faceValue, 1);

      // If player balance is less than face value, auto-set wager to face value
      if (playerBalance < faceValue) {
        this.wagerAmount = faceValue;
        this.currentWager = faceValue;
        this.currentQuestion = questionData;
        this.autoWager = true; // Flag to show auto-wager message
        this.fetchAndDisplayQuestion(questionData);
      } else {
        // Show wager input
        this.wagerAmount = faceValue;
        this.currentQuestion = questionData;
        this.autoWager = false;
        this.showDoubleJeopardyWager = true;
      }
    },

    submitWager() {
      if (!this.isValidWager) return;

      this.currentWager = this.wagerAmount;
      this.showDoubleJeopardyWager = false;
      this.fetchAndDisplayQuestion(this.currentQuestion);
    },

    async fetchAndDisplayQuestion(questionData) {
      try {
        // Ensure we respect the API rate limit
        const timeSinceLastCall = Date.now() - this.lastApiCall;
        if (timeSinceLastCall < this.apiDelay) {
          const waitTime = this.apiDelay - timeSinceLastCall;
          await new Promise((resolve) => setTimeout(resolve, waitTime));
        }

        this.loading = true;

        // Fetch question from API
        const response = await fetch(
          `https://opentdb.com/api.php?amount=1&category=${questionData.categoryId}&difficulty=${questionData.difficulty}&type=boolean`
        );
        const data = await response.json();

        this.lastApiCall = Date.now();

        if (data.response_code === 0 && data.results.length > 0) {
          const question = data.results[0];
          const decodedQuestion = this.decodeHtml(question.question);

          this.currentQuestion = {
            ...questionData,
            question: decodedQuestion,
            correctAnswer: question.correct_answer,
          };

          this.playerAnswer = "";
        } else {
          throw new Error("No questions available");
        }
      } catch (error) {
        console.error("Error fetching question:", error);
        alert("Error loading question. Please try another.");
      } finally {
        this.loading = false;
      }
    },

    submitAnswer() {
      if (!this.playerAnswer || !this.currentQuestion) return;

      const isCorrect =
        this.playerAnswer === this.currentQuestion.correctAnswer;
      const questionValue = this.isDoubleJeopardy
        ? this.currentWager
        : this.currentQuestion.value;

      // Update game board
      const categoryIndex = this.currentQuestion.categoryIndex;
      const questionIndex = this.currentQuestion.questionIndex;

      this.gameBoard[categoryIndex][questionIndex].answered = true;
      this.gameBoard[categoryIndex][questionIndex].correct = isCorrect;
      this.gameBoard[categoryIndex][questionIndex].playerId =
        this.currentPlayer;

      // Update player score
      if (isCorrect) {
        this.playerScores[this.currentPlayer - 1] += questionValue;
        this.resultMessage = this.isDoubleJeopardy
          ? `Correct! You won $${questionValue} in Double Jeopardy!`
          : "Correct!";
        this.nextPlayerMessage = `Player ${this.currentPlayer} to select`;
      } else {
        this.playerScores[this.currentPlayer - 1] -= questionValue;
        this.resultMessage = this.isDoubleJeopardy
          ? `Incorrect! You lost $${questionValue} in Double Jeopardy!`
          : "Incorrect!";
        this.currentPlayer =
          this.currentPlayer === this.numPlayers ? 1 : this.currentPlayer + 1;
        this.nextPlayerMessage = `Player ${this.currentPlayer} to select`;
      }

      // Reset Double Jeopardy state
      this.isDoubleJeopardy = false;
      this.currentWager = 0;

      // Update game state
      this.questionsAnswered++;
      this.currentQuestion = null;
      this.playerAnswer = "";
      this.showResult = true;

      // Check if game is over
      if (this.questionsAnswered >= this.totalQuestions) {
        setTimeout(() => {
          this.gameOver = true;
          this.showResult = false;
        }, 2000);
      } else {
        // Hide result after 3 seconds
        setTimeout(() => {
          this.showResult = false;
        }, 3000);
      }
    },

    decodeHtml(html) {
      const txt = document.createElement("textarea");
      txt.innerHTML = html;
      return txt.value;
    },
  },
};
</script>

<style scoped>
#app {
  font-family: Arial, sans-serif;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

h1 {
  text-align: center;
  color: #1e3a8a;
  margin-bottom: 30px;
}

.game-config {
  background-color: #f3f4f6;
  padding: 30px;
  border-radius: 12px;
  text-align: center;
  margin-bottom: 30px;
  border: 2px solid #1e3a8a;
}

.config-section {
  margin: 20px 0;
}

.config-section label {
  display: inline-block;
  width: 200px;
  text-align: right;
  margin-right: 10px;
  font-weight: bold;
  color: #1e3a8a;
}

.config-input {
  padding: 8px 12px;
  border: 2px solid #1e3a8a;
  border-radius: 4px;
  font-size: 16px;
  width: 80px;
}

.game-board {
  margin-bottom: 30px;
}

.board-table {
  width: 100%;
  border-collapse: collapse;
  border: 3px solid #1e3a8a;
}

.category-header {
  background-color: #1e3a8a;
  color: white;
  padding: 15px;
  text-align: center;
  font-size: 16px;
  font-weight: bold;
  border: 2px solid #000;
}

.question-cell {
  border: 2px solid #000;
  text-align: center;
  padding: 0;
  height: 80px;
}

.question-button {
  width: 100%;
  height: 100%;
  background-color: #3b82f6;
  color: #fbbf24;
  border: none;
  font-size: 24px;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.2s;
}

.question-button:hover {
  background-color: #2563eb;
}

.answered-cell {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  font-weight: bold;
  color: white;
}

.answered-cell.correct {
  background-color: #16a34a;
}

.answered-cell.incorrect {
  background-color: #dc2626;
}

.double-jeopardy-wager {
  background-color: #fbbf24;
  color: #1e3a8a;
  padding: 25px;
  border-radius: 12px;
  margin: 20px 0;
  text-align: center;
  border: 3px solid #f59e0b;
}

.double-jeopardy-wager h3 {
  margin-top: 0;
  font-size: 24px;
}

.wager-input {
  margin: 20px 0;
}

.wager-field {
  padding: 8px 12px;
  border: 2px solid #1e3a8a;
  border-radius: 4px;
  font-size: 16px;
  width: 100px;
  margin: 0 10px;
}

.wager-btn {
  background-color: #1e3a8a;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  font-weight: bold;
  cursor: pointer;
  margin-left: 10px;
}

.wager-btn:disabled {
  background-color: #6b7280;
  cursor: not-allowed;
}

.auto-wager-message {
  font-size: 14px;
  margin-top: 5px;
  opacity: 0.9;
}

.question-area {
  background-color: #3b82f6;
  color: white;
  padding: 20px;
  border-radius: 8px;
  margin: 20px 0;
}

.double-jeopardy-indicator {
  background-color: #fbbf24;
  color: #1e3a8a;
  padding: 10px;
  border-radius: 6px;
  text-align: center;
  font-weight: bold;
  margin-bottom: 15px;
}

.question-header {
  font-size: 18px;
  font-weight: bold;
  margin-bottom: 15px;
  color: #fbbf24;
}

.question-text {
  font-size: 16px;
  margin-bottom: 20px;
  line-height: 1.5;
}

.answer-buttons {
  display: flex;
  align-items: center;
  gap: 20px;
}

.answer-buttons label {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 16px;
  cursor: pointer;
}

.submit-btn {
  background-color: #fbbf24;
  color: #1e3a8a;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  font-weight: bold;
  cursor: pointer;
}

.submit-btn:disabled {
  background-color: #6b7280;
  cursor: not-allowed;
}

.result-area {
  background-color: #3b82f6;
  color: white;
  padding: 20px;
  border-radius: 8px;
  margin: 20px 0;
  text-align: center;
}

.result-message {
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 10px;
  color: #fbbf24;
}

.next-player {
  font-size: 16px;
  color: #fbbf24;
}

.game-over {
  background-color: #16a34a;
  color: white;
  padding: 30px;
  border-radius: 8px;
  text-align: center;
  margin: 20px 0;
}

.scores {
  background-color: #1e3a8a;
  color: white;
  padding: 20px;
  border-radius: 8px;
  margin: 20px 0;
  text-align: center;
}

.scores h3 {
  margin-top: 0;
  margin-bottom: 15px;
  color: #fbbf24;
}

.scores-horizontal {
  display: flex;
  justify-content: center;
  gap: 20px;
  flex-wrap: wrap;
}

.score {
  font-size: 16px;
  font-weight: bold;
  color: white;
  background-color: #2563eb;
  padding: 10px 15px;
  border-radius: 8px;
  min-width: 100px;
}

.loading {
  text-align: center;
  font-size: 18px;
  color: #1e3a8a;
  margin: 20px 0;
}

.start-btn {
  background-color: #1e3a8a;
  color: white;
  border: none;
  padding: 15px 30px;
  font-size: 18px;
  border-radius: 8px;
  cursor: pointer;
  margin-top: 20px;
}

.start-btn:hover {
  background-color: #1e40af;
}
</style>
