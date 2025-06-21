<template>
  <div id="app">
    <h1>Jeopardy</h1>

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

    <!-- Question Display Area -->
    <div v-if="currentQuestion" class="question-area">
      <p class="question-header">
        Player {{ currentPlayer }} selects
        {{ selectedCategories[currentQuestion.categoryIndex].name }} for ${{
          currentQuestion.value
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

    <!-- Start Game Button (for initial load) -->
    <button
      v-if="!gameStarted && !loading"
      @click="initializeGame"
      class="start-btn"
    >
      Start Game
    </button>
  </div>
</template>

<script>
export default {
  name: "JeopardyGame",
  data() {
    return {
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
      totalQuestions: 20, // 4 categories × 5 questions

      // Current game state
      currentPlayer: 1,
      playerScores: [0, 0, 0], // 3 players
      currentQuestion: null,
      playerAnswer: "",
      showResult: false,
      resultMessage: "",
      nextPlayerMessage: "",

      // API timing
      lastApiCall: 0,
      apiDelay: 5000, // 5 seconds between calls
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
  },

  methods: {
    async initializeGame() {
      this.loading = true;
      try {
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

        // Randomly select 4 categories
        this.selectedCategories = this.getRandomCategories(4);
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

      for (let categoryIndex = 0; categoryIndex < 4; categoryIndex++) {
        const categoryColumn = [];
        const category = this.selectedCategories[categoryIndex];

        // Setup questions for this category
        for (let questionIndex = 0; questionIndex < 5; questionIndex++) {
          const value = (questionIndex + 1) * 100; // $100, $200, $300, $400, $500
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

    async selectQuestion(categoryIndex, questionIndex) {
      if (this.currentQuestion || this.showResult) return;

      const questionData = this.gameBoard[categoryIndex][questionIndex];
      if (questionData.answered) return;

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

          // Decode HTML entities
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
      const questionValue = this.currentQuestion.value;

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
        this.resultMessage = "Correct!";
        this.nextPlayerMessage = `Player ${this.currentPlayer} to select`;
      } else {
        this.playerScores[this.currentPlayer - 1] -= questionValue;
        this.resultMessage = "Incorrect!";
        this.currentPlayer =
          this.currentPlayer === 3 ? 1 : this.currentPlayer + 1;
        this.nextPlayerMessage = `Player ${this.currentPlayer} to select`;
      }

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

.question-area {
  background-color: #3b82f6;
  color: white;
  padding: 20px;
  border-radius: 8px;
  margin: 20px 0;
  margin-right: 250px; /* Make space for scores panel */
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
  margin-right: 220px; /* Make space for scores */
}

.scores {
  background-color: #1e3a8a;
  color: white;
  padding: 20px;
  border-radius: 8px;
  position: fixed;
  bottom: 20px;
  right: 20px;
  min-width: 200px;
  z-index: 10; /* Ensure it stays on top */
}

.scores h3 {
  margin-top: 0;
  color: #fbbf24;
}

.score {
  margin: 8px 0;
  font-size: 16px;
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
  display: block;
  margin: 20px auto;
}

.start-btn:hover {
  background-color: #1e40af;
}
</style>
