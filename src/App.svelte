<script>
  import { onDestroy, tick } from 'svelte';
  import questionsData from './questions.json';

  const MAX_LIVES = 3;
  const QUESTION_TIME_LIMIT = 10;
  const totalQuestions = questionsData.length;

  // Game state
  let lives = MAX_LIVES;
  let remainingQuestions = [...questionsData];
  let currentQuestion = null;
  let userAnswer = '';
  let feedbackMessage = '';
  let feedbackStatus = ''; // 'correct' | 'wrong' | ''
  let gameState = 'idle'; // 'idle' | 'playing' | 'won' | 'lost'
  let correctCount = 0;
  let incorrectCount = 0;
  let inputField;

  // Timer state
  let timeLeft = QUESTION_TIME_LIMIT;
  let timerId = null;

  // Reactive count of total questions attempted/completed
  $: attemptedCount = correctCount + incorrectCount;

  // Pick a random question object from the given array
  function getRandomQuestion(questionList) {
    if (questionList.length === 0) {
      return null;
    }
    const randomIndex = Math.floor(Math.random() * questionList.length);
    return questionList[randomIndex];
  }

  // Stop any running countdown interval
  function stopTimer() {
    if (timerId !== null) {
      clearInterval(timerId);
      timerId = null;
    }
  }

  // Start or reset the 10-second countdown for the active question
  function startTimer() {
    stopTimer();
    timeLeft = QUESTION_TIME_LIMIT;

    timerId = setInterval(() => {
      if (gameState !== 'playing') {
        stopTimer();
        return;
      }

      timeLeft -= 1;

      if (timeLeft <= 0) {
        handleTimeOut();
      }
    }, 1000);
  }

  // Start the game for the first time when the Start button is pressed
  async function startGame() {
    correctCount = 0;
    incorrectCount = 0;
    remainingQuestions = [...questionsData];
    currentQuestion = getRandomQuestion(remainingQuestions);
    gameState = 'playing';
    startTimer();

    await tick();
    if (inputField) {
      inputField.focus();
    }
  }

  // Evaluate the player's answer (both manual submit and timer expiration)
  function evaluateAnswer(cleanedAnswer, isTimeout) {
    stopTimer();

    const isCorrect =
      cleanedAnswer !== '' &&
      cleanedAnswer === currentQuestion.answer.trim();

    if (isCorrect) {
      correctCount += 1;
      remainingQuestions = remainingQuestions.filter(
        (item) => item !== currentQuestion
      );

      if (remainingQuestions.length === 0) {
        gameState = 'won';
        return;
      }

      feedbackMessage = isTimeout
        ? "Time's up, but your answer was correct!"
        : 'Correct!';
      feedbackStatus = 'correct';
    } else {
      incorrectCount += 1;
      lives -= 1;

      if (lives <= 0) {
        lives = 0;
        gameState = 'lost';
        return;
      }

      if (isTimeout && cleanedAnswer === '') {
        feedbackMessage = `Time's up! No answer was entered for "${currentQuestion.question}".`;
      } else if (isTimeout) {
        feedbackMessage = `Time's up! "${currentQuestion.question}" is not ${cleanedAnswer}.`;
      } else {
        feedbackMessage = `Wrong! "${currentQuestion.question}" is not ${cleanedAnswer}.`;
      }
      feedbackStatus = 'wrong';
    }

    userAnswer = '';
    currentQuestion = getRandomQuestion(remainingQuestions);
    startTimer();

    if (inputField) {
      inputField.focus();
    }
  }

  // Check whatever is currently in the input box when the timer hits 0
  function handleTimeOut() {
    if (gameState !== 'playing') {
      return;
    }
    evaluateAnswer(userAnswer.trim(), true);
  }

  // Handle manual form submission
  function handleSubmit() {
    const cleanedAnswer = userAnswer.trim();

    if (cleanedAnswer === '' || gameState !== 'playing') {
      return;
    }

    evaluateAnswer(cleanedAnswer, false);
  }

  // Reset all fields and restart the timer for a fresh game
  async function restartGame() {
    lives = MAX_LIVES;
    correctCount = 0;
    incorrectCount = 0;
    remainingQuestions = [...questionsData];
    currentQuestion = getRandomQuestion(remainingQuestions);
    userAnswer = '';
    feedbackMessage = '';
    feedbackStatus = '';
    gameState = 'playing';
    startTimer();

    await tick();
    if (inputField) {
      inputField.focus();
    }
  }

  onDestroy(() => {
    stopTimer();
  });
</script>

<main class="game-container">
  <div class="card">
    <header class="status-bar">
      <div class="lives" aria-label="Remaining lives">
        {#each Array(MAX_LIVES) as _, index}
          <span class="heart" class:lost={index >= lives}>
            {index < lives ? '❤️' : '🖤'}
          </span>
        {/each}
      </div>

      {#if gameState === 'playing'}
        <div
          class="timer"
          class:urgent={timeLeft <= 3}
          aria-label="Time remaining"
        >
          ⏱️ {timeLeft}s
        </div>
      {/if}

      <div class="progress">
        Completed: <strong>{attemptedCount} / {totalQuestions}</strong>
      </div>
    </header>

    {#if gameState === 'idle'}
      <section class="start-screen">
        <h2>Subtraction Challenge</h2>
        <p>You have {QUESTION_TIME_LIMIT} seconds per question and {MAX_LIVES} lives in total.</p>
        <button on:click={startGame} class="btn primary">Start</button>
      </section>
    {:else if gameState === 'playing' && currentQuestion}
      <section class="play-area">
        <p class="prompt-label">Solve the subtraction:</p>
        <h1 class="question-text">{currentQuestion.question} = ?</h1>

        <form on:submit|preventDefault={handleSubmit} class="answer-form">
          <label for="answer-input" class="sr-only">Your Answer</label>
          <input
            id="answer-input"
            bind:this={inputField}
            bind:value={userAnswer}
            type="text"
            inputmode="numeric"
            placeholder="Enter answer"
            autocomplete="off"
            required
          />
          <button type="submit" class="btn primary">Submit</button>
        </form>

        <p class="feedback {feedbackStatus}" role="status" aria-live="polite">
          {feedbackMessage || '\u00A0'}
        </p>
      </section>
    {:else if gameState === 'won'}
      <section class="end-screen">
        <h1>You Won!</h1>
        <p>You answered all {totalQuestions} questions correctly.</p>
        <button on:click={restartGame} class="btn primary">Play Again</button>
      </section>
    {:else if gameState === 'lost'}
      <section class="end-screen">
        <h1>Game Over!</h1>
        <p class="end-subtitle">You ran out of lives.</p>

        <div class="stats-list" aria-label="Game summary">
          <div class="stat-row">
            <span class="stat-label">Questions Attempted / Completed</span>
            <span class="stat-value">{attemptedCount} / {totalQuestions}</span>
          </div>
          <div class="stat-row">
            <span class="stat-label">Answered Correctly</span>
            <span class="stat-value correct">{correctCount}</span>
          </div>
          <div class="stat-row">
            <span class="stat-label">Answered Incorrectly</span>
            <span class="stat-value wrong">{incorrectCount}</span>
          </div>
        </div>

        <button on:click={restartGame} class="btn primary">Try Again</button>
      </section>
    {/if}
  </div>
</main>

<style>
  :global(*),
  :global(*::before),
  :global(*::after) {
    box-sizing: border-box;
  }

  :global(html),
  :global(body) {
    margin: 0;
    padding: 0;
    width: 100%;
    height: 100%;
  }

  .game-container {
    min-height: 100vh;
    min-height: 100dvh;
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: #f4f6f9;
    font-family: system-ui, -apple-system, sans-serif;
    padding: clamp(0.5rem, 2.5vw, 1rem);
  }

  .card {
    background: #ffffff;
    width: 100%;
    max-width: 560px;
    padding: clamp(0.875rem, 4vw, 2rem);
    border-radius: 12px;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.08);
    text-align: center;
  }

  .status-bar {
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
    align-items: center;
    gap: 0.75rem;
    padding-bottom: 1.25rem;
    margin-bottom: 1.5rem;
    border-bottom: 1px solid #e5e7eb;
  }

  .lives {
    display: flex;
    gap: 0.35rem;
    font-size: 1.35rem;
  }

  .heart.lost {
    opacity: 0.35;
  }

  .timer {
    font-size: 1rem;
    font-weight: 700;
    color: #1f2937;
    background-color: #f3f4f6;
    padding: 0.3rem 0.7rem;
    border-radius: 999px;
    font-variant-numeric: tabular-nums;
  }

  .timer.urgent {
    color: #dc2626;
    background-color: #fee2e2;
  }

  .progress {
    font-size: 0.95rem;
    color: #4b5563;
  }

  .prompt-label {
    color: #6b7280;
    margin-bottom: 0.8rem;
    font-size: 0.95rem;
  }

  .question-text {
    font-size: clamp(2rem, 6vw, 2.5rem);
    margin: 0.5rem 0 1.5rem;
    color: #111827;
  }

  .answer-form {
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
    justify-content: center;
  }

  input {
    flex: 1 1 140px;
    max-width: 180px;
    min-width: 0;
    padding: 0.75rem;
    font-size: 1.25rem;
    text-align: center;
    border: 2px solid #d1d5db;
    border-radius: 8px;
    outline: none;
  }

  input:focus {
    border-color: #2563eb;
  }

  .btn {
    padding: 0.75rem 1.25rem;
    font-size: 1rem;
    font-weight: 600;
    border: none;
    border-radius: 8px;
    cursor: pointer;
  }

  .btn.primary {
    background-color: #2563eb;
    color: #ffffff;
  }

  .btn.primary:hover {
    background-color: #1d4ed8;
  }

  .feedback {
    margin: 1.25rem 0 0;
    min-height: 2.25rem;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: 600;
    font-size: 0.95rem;
    line-height: 1.4;
  }

  .feedback.correct {
    color: #16a34a;
  }

  .feedback.wrong {
    color: #dc2626;
  }

  .start-screen h2,
  .end-screen h1 {
    margin-top: 0;
    margin-bottom: 1.25rem;
    color: #111827;
  }

  .start-screen p,
  .end-screen p {
    color: #4b5563;
    margin-bottom: 1.5rem;
  }

  .start-screen p {
    font-size: clamp(0.68rem, 3vw, 1rem);
  }

  .end-screen .end-subtitle {
    margin-bottom: 1rem;
  }

  .stats-list {
    display: flex;
    flex-direction: column;
    gap: 0.6rem;
    max-width: 380px;
    margin: 0 auto 1.5rem;
    padding: 0.9rem 1.1rem;
    background-color: #f9fafb;
    border: 1px solid #e5e7eb;
    border-radius: 10px;
  }

  .stat-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 1rem;
    font-size: 0.95rem;
  }

  .stat-label {
    color: #4b5563;
    text-align: left;
  }

  .stat-value {
    font-weight: 700;
    color: #111827;
    font-variant-numeric: tabular-nums;
  }

  .stat-value.correct {
    color: #16a34a;
  }

  .stat-value.wrong {
    color: #dc2626;
  }

  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
  }
</style>
