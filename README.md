import React, { useState, useEffect, createContext, useContext, useReducer } from 'react';

// Constants for screen names
const SCREENS = {
  MAIN_MENU: 'MAIN_MENU',
  TEAM_SETUP: 'TEAM_SETUP',
  CATEGORY_SELECTION: 'CATEGORY_SELECTION',
  QUESTION_SELECT: 'QUESTION_SELECT',
  QUESTION_DISPLAY: 'QUESTION_DISPLAY',
  ANSWER_REVIEW: 'ANSWER_REVIEW',
  STORE: 'STORE',
  END_GAME: 'END_GAME',
  BONUS_CHALLENGE_SETUP: 'BONUS_CHALLENGE_SETUP',
  BONUS_CHALLENGE_QUESTION: 'BONUS_CHALLENGE_QUESTION',
  WAGER_SETUP: 'WAGER_SETUP',
};


// Game Context
const GameContext = createContext();

const initialState = {
  currentScreen: SCREENS.MAIN_MENU,
  theme: 'light', // 'light' or 'dark'
  tokens: 10, // Starting tokens
  teams: [
    { id: 1, name: 'Team 1', members: [], icon: '🏆', score: 0, wagerUsed: false, memberCount: 0 },
    { id: 2, name: 'Team 2', members: [], icon: '🏅', score: 0, wagerUsed: false, memberCount: 0 },
  ],
  selectedCategories: [], // Array of category IDs
  currentQuestion: null, // { categoryId, questionId, questionObj }
  answeredQuestions: {}, // { 'catId_qId': true }
  activeTeamIndex: 0, // To track whose turn it might be (though not strictly turn-based for all actions)
  wagerState: null, // { wageringTeamId, targetTeamId, question: null, multiplier: 1 }
  gameMessage: null, // For temporary messages like "Wager Question for Team X"
};

function gameReducer(state, action) {
  switch (action.type) {
    case 'NAVIGATE':
      return { ...state, currentScreen: action.payload, gameMessage: null }; // Clear message on navigation
    case 'TOGGLE_THEME':
      return { ...state, theme: state.theme === 'light' ? 'dark' : 'light' };
    case 'SET_TOKENS':
      return { ...state, tokens: action.payload };
    case 'UPDATE_TEAM_NAME':
      return {
        ...state,
        teams: state.teams.map(team =>
          team.id === action.payload.teamId ? { ...team, name: action.payload.name } : team
        ),
      };
    case 'UPDATE_TEAM_MEMBER_COUNT':
      return {
        ...state,
        teams: state.teams.map(team =>
          team.id === action.payload.teamId ? { ...team, memberCount: action.payload.count } : team
        ),
      };
    case 'UPDATE_TEAM_ICON':
       return {
        ...state,
        teams: state.teams.map(team =>
          team.id === action.payload.teamId ? { ...team, icon: action.payload.icon } : team
        ),
      };
    case 'SELECT_CATEGORY':
      const newSelected = state.selectedCategories.includes(action.payload)
        ? state.selectedCategories.filter(id => id !== action.payload)
        : [...state.selectedCategories, action.payload];
      return { ...state, selectedCategories: newSelected.slice(0, 6) };
    case 'START_GAME':
      if (state.tokens < 1) return { ...state, gameMessage: "Not enough tokens!" };
      return { 
        ...state, 
        tokens: state.tokens - 1, 
        currentScreen: SCREENS.QUESTION_SELECT, 
        answeredQuestions: {}, // Reset answered questions for new game
        teams: state.teams.map(t => ({...t, score: 0, wagerUsed: false})), // Reset scores and wagers
        wagerState: null,
      };
    case 'SELECT_QUESTION':
      return { ...state, currentQuestion: action.payload, currentScreen: SCREENS.QUESTION_DISPLAY };
    case 'MARK_ANSWERED':
      const { categoryId, questionId } = action.payload;
      return {
        ...state,
        answeredQuestions: {
          ...state.answeredQuestions,
          [`${categoryId}_${questionId}`]: true,
        },
        currentScreen: SCREENS.QUESTION_SELECT,
        currentQuestion: null, // Clear current question after answering
      };
    case 'AWARD_POINTS':
      const { teamId, points } = action.payload;
      return {
        ...state,
        teams: state.teams.map(team =>
          team.id === teamId ? { ...team, score: team.score + points } : team
        ),
      };
    case 'ADJUST_POINTS':
      return {
        ...state,
        teams: state.teams.map(team =>
          team.id === action.payload.teamId ? { ...team, score: Math.max(0, team.score + action.payload.amount) } : team
        ),
      };
    case 'INITIATE_WAGER_SETUP':
      return {
        ...state,
        wagerState: { wageringTeamId: action.payload.wageringTeamId, targetTeamId: null, question: null, multiplier: 1 },
        currentScreen: SCREENS.WAGER_SETUP,
      };
    case 'SET_WAGER_TARGET':
      // Find a random unanswered question for the target team
      const availableCategoriesForWager = state.selectedCategories.map(catId => MOCK_DATABASE.categories.find(c => c.id === catId));
      let wagerQuestionPool = [];
      availableCategoriesForWager.forEach(cat => {
        if (cat) {
          cat.questions.forEach(q => {
            if (!state.answeredQuestions[`${cat.id}_${q.id}`]) {
              wagerQuestionPool.push({ categoryId: cat.id, questionId: q.id, questionObj: q });
            }
          });
        }
      });

      if (wagerQuestionPool.length === 0) {
        return { ...state, gameMessage: "No questions left for wager!", currentScreen: SCREENS.QUESTION_SELECT, wagerState: null };
      }
      
      const randomWagerQuestion = wagerQuestionPool[Math.floor(Math.random() * wagerQuestionPool.length)];
      const multipliers = [0.5, 1.5, 2];
      const randomMultiplier = multipliers[Math.floor(Math.random() * multipliers.length)];

      return {
        ...state,
        wagerState: { 
          ...state.wagerState, 
          targetTeamId: action.payload.targetTeamId,
          question: randomWagerQuestion,
          multiplier: randomMultiplier,
        },
        currentQuestion: randomWagerQuestion, // Set this as the current question
        gameMessage: `WAGER! Team ${state.teams.find(t => t.id === action.payload.targetTeamId)?.name} to answer. Multiplier: ${randomMultiplier}x`,
        currentScreen: SCREENS.QUESTION_DISPLAY, // Go to question display
        teams: state.teams.map(t => t.id === state.wagerState.wageringTeamId ? {...t, wagerUsed: true} : t)
      };
    case 'RESOLVE_WAGER':
      const { wagerAwardTeamId, correct } = action.payload;
      const wager = state.wagerState;
      if (!wager || !wager.question) return state; // Should not happen

      let pointsToAward = 0;
      const basePoints = wager.question.questionObj.points;

      if (wager.multiplier === 0.5) {
        pointsToAward = correct ? basePoints * 0.5 : -basePoints * 0.5;
      } else if (wager.multiplier === 1.5) {
        pointsToAward = correct ? basePoints * 1.5 : -basePoints * 1;
      } else if (wager.multiplier === 2) {
        pointsToAward = correct ? basePoints * 2 : -basePoints * 1.5;
      }
      
      // Mark wager question as answered
      const newAnswered = {
        ...state.answeredQuestions,
        [`${wager.question.categoryId}_${wager.question.questionId}`]: true,
      };

      return {
        ...state,
        teams: state.teams.map(team =>
          team.id === wagerAwardTeamId ? { ...team, score: team.score + pointsToAward } : team
        ),
        answeredQuestions: newAnswered,
        wagerState: null,
        currentQuestion: null,
        currentScreen: SCREENS.QUESTION_SELECT,
        gameMessage: null,
      };
    case 'CLEAR_WAGER':
      return {...state, wagerState: null, gameMessage: null, currentScreen: SCREENS.QUESTION_SELECT };
    case 'SET_GAME_MESSAGE':
      return { ...state, gameMessage: action.payload };
    case 'RESET_GAME':
      return {
        ...initialState,
        tokens: state.tokens, // Keep tokens
        theme: state.theme, // Keep theme
        // Reset teams but keep names and icons if desired, or full reset:
        teams: initialState.teams.map((t, i) => ({ ...t, name: state.teams[i]?.name || `Team ${i+1}`, icon: state.teams[i]?.icon || (i === 0 ? '🏆' : '🏅'), memberCount: state.teams[i]?.memberCount || 0 })),
      };
    case 'START_BONUS_CHALLENGE_SETUP':
      return {...state, currentScreen: SCREENS.BONUS_CHALLENGE_SETUP };
    case 'START_BONUS_ROUND':
      // action.payload should be an array of two selected category IDs for bonus
      const bonusCategories = action.payload.map(catId => MOCK_DATABASE.categories.find(c => c.id === catId));
      let bonusQuestions = [];
      if (bonusCategories.length === 2) {
        // Assign one random question from each chosen category
        const cat1QuestionPool = bonusCategories[0].questions.filter(q => !state.answeredQuestions[`${bonusCategories[0].id}_${q.id}`]);
        if (cat1QuestionPool.length > 0) {
            bonusQuestions.push({ teamId: state.teams[0].id, category: bonusCategories[0], question: cat1QuestionPool[Math.floor(Math.random() * cat1QuestionPool.length)] });
        }
        const cat2QuestionPool = bonusCategories[1].questions.filter(q => !state.answeredQuestions[`${bonusCategories[1].id}_${q.id}`]);
        if (cat2QuestionPool.length > 0) {
            bonusQuestions.push({ teamId: state.teams[1].id, category: bonusCategories[1], question: cat2QuestionPool[Math.floor(Math.random() * cat2QuestionPool.length)] });
        }
      }
      if (bonusQuestions.length === 0) {
        return {...state, gameMessage: "Not enough unique questions for bonus round.", currentScreen: SCREENS.END_GAME };
      }
      return {...state, currentScreen: SCREENS.BONUS_CHALLENGE_QUESTION, bonusRound: { questions: bonusQuestions, currentIndex: 0, answers: [] }};
    case 'ANSWER_BONUS_QUESTION':
      // action.payload: { teamId, question, correct }
      const currentBonus = state.bonusRound;
      const bonusPoints = action.payload.question.points * (action.payload.correct ? 2 : -1);
      const updatedTeams = state.teams.map(t => t.id === action.payload.teamId ? {...t, score: t.score + bonusPoints} : t);
      const newBonusAnswers = [...currentBonus.answers, action.payload];
      
      // Mark bonus question as answered
      const newAnsweredBonus = {
        ...state.answeredQuestions,
        [`${action.payload.category.id}_${action.payload.question.id}`]: true,
      };

      if (newBonusAnswers.length === currentBonus.questions.length) {
        // Bonus round finished
        return {...state, teams: updatedTeams, answeredQuestions: newAnsweredBonus, bonusRound: null, currentScreen: SCREENS.END_GAME, gameMessage: "Bonus Round Complete!"};
      } else {
        return {...state, teams: updatedTeams, answeredQuestions: newAnsweredBonus, bonusRound: {...currentBonus, currentIndex: currentBonus.currentIndex + 1, answers: newBonusAnswers }};
      }

    default:
      return state;
  }
}

// --- Components ---

// Utility Button Component
const Button = ({ onClick, children, className = '', variant = 'primary', size = 'md', disabled = false }) => {
  const { theme } = useContext(GameContext);
  const baseStyle = `font-semibold rounded-lg shadow-md focus:outline-none focus:ring-2 focus:ring-opacity-75 transition-all duration-150 ease-in-out`;
  const sizeStyles = {
    sm: 'px-3 py-1.5 text-sm',
    md: 'px-4 py-2 text-base',
    lg: 'px-6 py-3 text-lg',
  };
  const variantStyles = {
    primary: theme === 'light' 
      ? 'bg-blue-600 hover:bg-blue-700 text-white focus:ring-blue-500' 
      : 'bg-blue-500 hover:bg-blue-600 text-white focus:ring-blue-400',
    secondary: theme === 'light' 
      ? 'bg-gray-200 hover:bg-gray-300 text-gray-800 focus:ring-gray-400' 
      : 'bg-gray-700 hover:bg-gray-600 text-gray-200 focus:ring-gray-500',
    danger: theme === 'light'
      ? 'bg-red-500 hover:bg-red-600 text-white focus:ring-red-400'
      : 'bg-red-600 hover:bg-red-700 text-white focus:ring-red-500',
    ghost: theme === 'light'
        ? 'bg-transparent hover:bg-gray-200 text-gray-700 focus:ring-gray-400'
        : 'bg-transparent hover:bg-gray-700 text-gray-200 focus:ring-gray-500',
  };
  const disabledStyle = disabled ? (theme === 'light' ? 'bg-gray-300 text-gray-500 cursor-not-allowed' : 'bg-gray-600 text-gray-400 cursor-not-allowed') : '';

  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`${baseStyle} ${sizeStyles[size]} ${variantStyles[variant]} ${disabled ? disabledStyle : ''} ${className}`}
    >
      {children}
    </button>
  );
};

const ThemeToggleButton = () => {
  const { theme, dispatch } = useContext(GameContext);
  return (
    <Button 
        onClick={() => dispatch({ type: 'TOGGLE_THEME' })} 
        variant="ghost"
        className="absolute top-4 right-4 p-2 rounded-full"
        aria-label="Toggle theme"
    >
      {theme === 'light' ? 
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" strokeWidth={1.5} stroke="currentColor" className="w-6 h-6 text-gray-700">
          <path strokeLinecap="round" strokeLinejoin="round" d="M21.752 15.002A9.72 9.72 0 0 1 18 15.75c-5.385 0-9.75-4.365-9.75-9.75 0-1.33.266-2.597.748-3.752A9.753 9.753 0 0 0 3 11.25C3 16.635 7.365 21 12.75 21a9.753 9.753 0 0 0 9.002-5.998Z" />
        </svg>
       : 
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" strokeWidth={1.5} stroke="currentColor" className="w-6 h-6 text-yellow-400">
          <path strokeLinecap="round" strokeLinejoin="round" d="M12 3v2.25m6.364.386-1.591 1.591M21 12h-2.25m-.386 6.364-1.591-1.591M12 18.75V21m-6.364-.386 1.591-1.591M3 12h2.25m.386-6.364 1.591 1.591M12 12a2.25 2.25 0 0 0-2.25 2.25c0 1.37.448 2.592 1.197 3.595A2.25 2.25 0 0 0 12 12Zm0 0a2.25 2.25 0 0 1 2.25 2.25c0 1.37-.448 2.592-1.197 3.595A2.25 2.25 0 0 1 12 12Z" />
        </svg>
      }
    </Button>
  );
};

const TokenDisplay = () => {
  const { state, dispatch } = useContext(GameContext);
  return (
    <div className={`absolute top-4 left-4 flex items-center p-2 rounded-full shadow-md ${state.theme === 'light' ? 'bg-amber-100 text-amber-700' : 'bg-gray-700 text-amber-300'}`}>
      <span className="font-bold text-lg mr-2">{state.tokens}</span>
      <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" className="w-5 h-5 mr-2">
        <path d="M10 1a9 9 0 1 0 0 18 9 9 0 0 0 0-18ZM8.05 4.95a.75.75 0 0 1 1.06-.04l4.25 3.5a.75.75 0 0 1 0 1.18l-4.25 3.5a.75.75 0 1 1-.98-1.14L11.7 10 8.1 7.52a.75.75 0 0 1-.04-1.06l.01-.01Z" />
      </svg>
      <span className="mr-3 font-medium">Tokens</span>
      <button 
        onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.STORE })}
        className={`p-1.5 rounded-full transition-colors ${state.theme === 'light' ? 'bg-green-500 hover:bg-green-600 text-white' : 'bg-green-600 hover:bg-green-700 text-white'}`}
        aria-label="Buy Tokens"
      >
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" strokeWidth={2} stroke="currentColor" className="w-5 h-5">
          <path strokeLinecap="round" strokeLinejoin="round" d="M12 4.5v15m7.5-7.5h-15" />
        </svg>
      </button>
    </div>
  );
};

const MainMenuScreen = () => {
  const { dispatch } = useContext(GameContext);
  return (
    <div className="flex flex-col items-center justify-center h-full p-8">
      <h1 className="text-5xl font-bold mb-12">Trivia Night</h1>
      <div className="space-y-6 w-full max-w-md">
        <Button onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.TEAM_SETUP })} size="lg" className="w-full">
          Play New Game
        </Button>
        <Button onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.STORE })} size="lg" variant="secondary" className="w-full">
          Token Store
        </Button>
         <Button onClick={() => dispatch({ type: 'RESET_GAME' })} size="lg" variant="secondary" className="w-full mt-8">
          Reset Game Data (Dev)
        </Button>
      </div>
    </div>
  );
};

const TeamSetupScreen = () => {
  const { state, dispatch } = useContext(GameContext);
  const preloadedIcons = ['🏆', '🏅', '🚀', '💡', '🎉', '🌟', '🎯', '🎮'];

  const handleNameChange = (teamId, name) => {
    dispatch({ type: 'UPDATE_TEAM_NAME', payload: { teamId, name } });
  };

  const handleMemberCountChange = (teamId, change) => {
    const team = state.teams.find(t => t.id === teamId);
    const newCount = Math.max(0, (team.memberCount || 0) + change);
    dispatch({ type: 'UPDATE_TEAM_MEMBER_COUNT', payload: { teamId, count: newCount } });
  };
  
  const handleIconChange = (teamId, icon) => {
    dispatch({ type: 'UPDATE_TEAM_ICON', payload: { teamId, icon } });
  };

  return (
    <div className="flex flex-col items-center justify-center h-full p-8">
      <h1 className="text-4xl font-bold mb-8">Team Setup</h1>
      <div className="grid grid-cols-1 md:grid-cols-2 gap-8 w-full max-w-4xl">
        {state.teams.map((team) => (
          <div key={team.id} className={`p-6 rounded-xl shadow-lg ${state.theme === 'light' ? 'bg-white' : 'bg-gray-700'}`}>
            <h2 className="text-2xl font-semibold mb-4">Team {team.id}</h2>
            <input
              type="text"
              value={team.name}
              onChange={(e) => handleNameChange(team.id, e.target.value)}
              placeholder="Enter Team Name"
              className={`w-full p-3 border rounded-lg mb-4 ${state.theme === 'light' ? 'border-gray-300 bg-gray-50 text-gray-900' : 'border-gray-600 bg-gray-800 text-white'}`}
            />
            
            <div className="mb-4">
              <label className="block text-sm font-medium mb-1">Team Icon:</label>
              <div className="flex space-x-2 items-center">
                <span className="text-3xl p-2 border rounded-lg">{team.icon}</span>
                <select 
                    value={team.icon} 
                    onChange={(e) => handleIconChange(team.id, e.target.value)}
                    className={`p-3 border rounded-lg ${state.theme === 'light' ? 'border-gray-300 bg-gray-50 text-gray-900' : 'border-gray-600 bg-gray-800 text-white'} flex-grow`}
                >
                    {preloadedIcons.map(icon => <option key={icon} value={icon}>{icon}</option>)}
                </select>
              </div>
            </div>

            <div className="mb-4">
              <label className="block text-sm font-medium mb-1">Team Members Count:</label>
              <div className="flex items-center space-x-2">
                <Button onClick={() => handleMemberCountChange(team.id, -1)} size="sm" variant="secondary">-</Button>
                <span className="text-xl font-semibold w-10 text-center">{team.memberCount}</span>
                <Button onClick={() => handleMemberCountChange(team.id, 1)} size="sm" variant="secondary">+</Button>
              </div>
            </div>
            {/* Optional: Input for team members' names - simplified for now */}
          </div>
        ))}
      </div>
      <div className="mt-12 flex space-x-4">
        <Button onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.MAIN_MENU })} variant="secondary" size="lg">
          Back to Menu
        </Button>
        <Button onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.CATEGORY_SELECTION })} size="lg">
          Next: Select Categories
        </Button>
      </div>
    </div>
  );
};

const CategorySelectionScreen = () => {
  const { state, dispatch } = useContext(GameContext);
  const { categories } = MOCK_DATABASE;

  const handleSelectCategory = (categoryId) => {
    dispatch({ type: 'SELECT_CATEGORY', payload: categoryId });
  };

  const startGame = () => {
    if (state.tokens < 1) {
        dispatch({ type: 'SET_GAME_MESSAGE', payload: "Not enough tokens to start! Visit the store." });
        // Optionally navigate to store: dispatch({ type: 'NAVIGATE', payload: SCREENS.STORE });
        return;
    }
    dispatch({ type: 'START_GAME' });
  };

  const canStartGame = state.selectedCategories.length === 6;

  return (
    <div className="flex flex-col items-center h-full p-4 pt-8 md:p-8">
      <h1 className="text-3xl md:text-4xl font-bold mb-2">Select 6 Categories</h1>
      <p className="mb-6 text-sm md:text-base">You have selected {state.selectedCategories.length} of 6 categories.</p>
      {state.gameMessage && <p className={`mb-4 text-center font-semibold ${state.theme === 'light' ? 'text-red-600' : 'text-red-400'}`}>{state.gameMessage}</p>}
      <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-4 gap-3 md:gap-4 overflow-y-auto flex-grow w-full max-w-5xl px-2">
        {categories.map((category) => (
          <button
            key={category.id}
            onClick={() => handleSelectCategory(category.id)}
            className={`relative p-3 md:p-4 rounded-2xl md:rounded-3xl aspect-square flex flex-col items-center justify-center text-center transition-all duration-200 group
                        ${state.selectedCategories.includes(category.id) 
                            ? (state.theme === 'light' ? 'bg-blue-500 text-white ring-4 ring-blue-300' : 'bg-blue-600 text-white ring-4 ring-blue-400') 
                            : (state.theme === 'light' ? 'bg-white hover:bg-gray-100 shadow-md' : 'bg-gray-700 hover:bg-gray-600 shadow-lg')}
                        ${state.selectedCategories.length >= 6 && !state.selectedCategories.includes(category.id) ? 'opacity-50 cursor-not-allowed' : ''}`}
            disabled={state.selectedCategories.length >= 6 && !state.selectedCategories.includes(category.id)}
          >
            <img src={category.image} alt={category.name} className="w-16 h-16 md:w-24 md:h-24 object-cover rounded-xl mb-2 md:mb-3 group-hover:scale-105 transition-transform" onError={(e) => e.target.src='https://placehold.co/100x100/cccccc/969696?text=Error'}/>
            <span className="text-xs md:text-sm font-semibold">{category.name}</span>
            {state.selectedCategories.includes(category.id) && (
              <div className={`absolute top-2 right-2 w-5 h-5 md:w-6 md:h-6 rounded-full flex items-center justify-center ${state.theme === 'light' ? 'bg-white text-blue-500' : 'bg-gray-800 text-blue-400'}`}>
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" className="w-4 h-4 md:w-5 md:h-5">
                  <path fillRule="evenodd" d="M16.704 4.153a.75.75 0 0 1 .143 1.052l-8 10.5a.75.75 0 0 1-1.127.075l-4.5-4.5a.75.75 0 0 1 1.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 0 1 1.05-.143Z" clipRule="evenodd" />
                </svg>
              </div>
            )}
          </button>
        ))}
      </div>
      <div className="mt-6 md:mt-8 flex space-x-4">
        <Button onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.TEAM_SETUP })} variant="secondary" size="lg">
          Back
        </Button>
        <Button onClick={startGame} size="lg" disabled={!canStartGame}>
          Start Game ({state.tokens} Token)
        </Button>
      </div>
    </div>
  );
};

const QuestionSelectScreen = () => {
  const { state, dispatch } = useContext(GameContext);
  const selectedCategoryDetails = state.selectedCategories.map(id => MOCK_DATABASE.categories.find(cat => cat.id === id)).filter(Boolean);

  const allQuestionsAnswered = () => {
    let totalQuestions = 0;
    let answeredCount = 0;
    selectedCategoryDetails.forEach(cat => {
        cat.questions.forEach(q => {
            totalQuestions++;
            if (state.answeredQuestions[`${cat.id}_${q.id}`]) {
                answeredCount++;
            }
        });
    });
    return totalQuestions > 0 && answeredCount === totalQuestions;
  };

  useEffect(() => {
    if (allQuestionsAnswered()) {
        dispatch({ type: 'NAVIGATE', payload: SCREENS.END_GAME });
    }
  }, [state.answeredQuestions, selectedCategoryDetails, dispatch]);


  const handleQuestionSelect = (category, question) => {
    dispatch({ type: 'SELECT_QUESTION', payload: { categoryId: category.id, questionId: question.id, questionObj: question } });
  };

  const handleRandomQuestion = () => {
    let available = [];
    selectedCategoryDetails.forEach(cat => {
      cat.questions.forEach(q => {
        if (!state.answeredQuestions[`${cat.id}_${q.id}`]) {
          available.push({ categoryId: cat.id, questionId: q.id, questionObj: q });
        }
      });
    });
    if (available.length > 0) {
      const randomQ = available[Math.floor(Math.random() * available.length)];
      handleQuestionSelect({id: randomQ.categoryId}, randomQ.questionObj);
    } else {
      dispatch({ type: 'SET_GAME_MESSAGE', payload: "All questions answered!" });
      dispatch({ type: 'NAVIGATE', payload: SCREENS.END_GAME });
    }
  };
  
  const getPointsForDifficulty = (difficulty) => {
    if (difficulty === 'easy') return 200;
    if (difficulty === 'medium') return 400;
    if (difficulty === 'hard') return 600;
    return 0;
  };

  return (
    <div className="flex flex-col h-full p-2 md:p-4 landscape-container">
      <h1 className="text-2xl md:text-3xl font-bold text-center my-3 md:my-4">Select a Question</h1>
      {state.gameMessage && <p className={`my-2 text-center font-semibold ${state.theme === 'light' ? 'text-indigo-600' : 'text-indigo-400'}`}>{state.gameMessage}</p>}
      
      <div className="grid grid-cols-3 gap-2 md:gap-4 flex-grow items-start w-full max-w-6xl mx-auto">
        {selectedCategoryDetails.map((category) => (
          <div key={category.id} className={`rounded-xl p-2 md:p-3 flex flex-col items-center ${state.theme === 'light' ? 'bg-white shadow-lg' : 'bg-gray-700 shadow-xl'}`}>
            <img src={category.image} alt={category.name} className="w-16 h-10 md:w-24 md:h-16 object-cover rounded-md mb-1 md:mb-2" onError={(e) => e.target.src='https://placehold.co/100x60/cccccc/969696?text=Error'}/>
            <h2 className="text-sm md:text-base font-semibold text-center mb-2 truncate w-full px-1">{category.name}</h2>
            <div className="grid grid-cols-2 gap-1 md:gap-2 w-full">
              {/* Filter questions to 3 difficulty tiers, assuming 2 per tier for 6 total */}
              {['easy', 'medium', 'hard'].flatMap(diff => 
                category.questions
                  .filter(q => q.difficulty === diff)
                  .slice(0, 2) // Take up to 2 questions per difficulty for this display
                  .map(q => {
                    const points = getPointsForDifficulty(q.difficulty); // Or use q.points
                    const isAnswered = state.answeredQuestions[`${category.id}_${q.id}`];
                    return (
                      <button
                        key={q.id}
                        onClick={() => handleQuestionSelect(category, q)}
                        className={`py-2 px-1 md:py-3 md:px-2 rounded-lg text-xs md:text-sm font-bold transition-all
                                    ${isAnswered 
                                        ? (state.theme === 'light' ? 'bg-gray-300 text-gray-500 hover:bg-gray-400' : 'bg-gray-600 text-gray-400 hover:bg-gray-500')
                                        : (state.theme === 'light' ? 'bg-yellow-400 hover:bg-yellow-500 text-yellow-900' : 'bg-yellow-500 hover:bg-yellow-600 text-yellow-100')
                                    } focus:outline-none focus:ring-2 ${state.theme === 'light' ? 'focus:ring-yellow-500' : 'focus:ring-yellow-300'}`}
                      >
                        {points}
                      </button>
                    );
                  })
              )}
            </div>
          </div>
        ))}
      </div>

      <Button onClick={handleRandomQuestion} size="lg" className="my-4 md:my-6 mx-auto">
        Random Question
      </Button>

      {/* Score Display - smaller and at the bottom */}
      <div className={`fixed bottom-0 left-0 right-0 p-2 md:p-3 shadow-top-md ${state.theme === 'light' ? 'bg-stone-100 border-t border-stone-200' : 'bg-gray-800 border-t border-gray-700'}`}>
        <div className="flex justify-around items-center max-w-3xl mx-auto">
          {state.teams.map(team => (
            <div key={team.id} className="flex flex-col items-center mx-1 md:mx-2">
              <div className="flex items-center">
                 <Button onClick={() => dispatch({ type: 'ADJUST_POINTS', payload: { teamId: team.id, amount: -50 } })} variant="ghost" size="sm" className="p-1 md:p-1.5">
                    <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor" className="w-4 h-4 md:w-5 md:h-5"><path strokeLinecap="round" strokeLinejoin="round" d="M5 12h14" /></svg>
                 </Button>
                <span className={`text-sm md:text-base font-semibold mx-1 md:mx-2 ${state.theme === 'light' ? 'text-gray-700' : 'text-gray-200'}`}>{team.icon} {team.name}:</span>
                <span className={`text-lg md:text-xl font-bold ${state.theme === 'light' ? 'text-blue-600' : 'text-blue-400'}`}>{team.score}</span>
                 <Button onClick={() => dispatch({ type: 'ADJUST_POINTS', payload: { teamId: team.id, amount: 50 } })} variant="ghost" size="sm" className="p-1 md:p-1.5">
                    <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor" className="w-4 h-4 md:w-5 md:h-5"><path strokeLinecap="round" strokeLinejoin="round" d="M12 4.5v15m7.5-7.5h-15" /></svg>
                 </Button>
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};


const QuestionDisplayScreen = () => {
  const { state, dispatch } = useContext(GameContext);
  const [timer, setTimer] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => setTimer(t => t + 1), 1000);
    return () => clearInterval(interval);
  }, []);

  if (!state.currentQuestion) {
    // Should not happen, but good to handle
    dispatch({ type: 'NAVIGATE', payload: SCREENS.QUESTION_SELECT });
    return null;
  }

  const { questionObj } = state.currentQuestion;
  const isWagerQuestion = state.wagerState && state.wagerState.question && state.wagerState.question.questionId === questionObj.id;
  const wagerTargetTeam = isWagerQuestion ? state.teams.find(t => t.id === state.wagerState.targetTeamId) : null;

  return (
    <div className="flex flex-col items-center justify-center h-full p-4 md:p-8">
      {isWagerQuestion && wagerTargetTeam && (
        <div className={`p-3 mb-4 rounded-lg text-center font-semibold text-lg ${state.theme === 'light' ? 'bg-red-100 text-red-700' : 'bg-red-900 text-red-300'}`}>
          WAGER QUESTION for {wagerTargetTeam.name} ONLY! Multiplier: {state.wagerState.multiplier}x
        </div>
      )}
      {state.gameMessage && !isWagerQuestion && <p className={`mb-4 text-center font-semibold ${state.theme === 'light' ? 'text-indigo-600' : 'text-indigo-400'}`}>{state.gameMessage}</p>}
      
      <div className={`p-6 md:p-10 rounded-xl shadow-2xl w-full max-w-3xl text-center ${state.theme === 'light' ? 'bg-white' : 'bg-gray-700'}`}>
        {questionObj.image && <img src={questionObj.image} alt="Question context" className="mx-auto mb-4 rounded-lg max-h-60" onError={(e) => e.target.style.display='none'} />}
        <p className="text-xl md:text-3xl font-semibold mb-6">{questionObj.text}</p>
        <p className="text-lg md:text-2xl mb-8">Points: {questionObj.points}</p>
        <div className={`text-2xl md:text-4xl font-mono mb-8 p-3 rounded-lg inline-block ${state.theme === 'light' ? 'bg-gray-100 text-gray-800' : 'bg-gray-600 text-gray-100'}`}>
          {Math.floor(timer / 60).toString().padStart(2, '0')}:{ (timer % 60).toString().padStart(2, '0') }
        </div>
        <Button onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.ANSWER_REVIEW })} size="lg" className="w-full md:w-auto px-10">
          Show Answer
        </Button>
      </div>
    </div>
  );
};

const AnswerReviewScreen = () => {
  const { state, dispatch } = useContext(GameContext);
  const { currentQuestion, teams, wagerState } = state;

  if (!currentQuestion) {
    // Should not happen
    dispatch({ type: 'NAVIGATE', payload: SCREENS.QUESTION_SELECT });
    return null;
  }

  const { questionObj, categoryId, questionId } = currentQuestion;
  const isWagerQuestion = wagerState && wagerState.question && wagerState.question.questionId === questionId;
  
  const handleAwardPoints = (teamId, correct) => {
    if (isWagerQuestion) {
      dispatch({ type: 'RESOLVE_WAGER', payload: { wagerAwardTeamId: teamId, correct: correct } });
    } else {
      const pointsToAward = correct ? questionObj.points : 0; // Standard play: no penalty for wrong
      if (teamId && pointsToAward > 0) { // Award points only if a team is selected and answer is correct
         dispatch({ type: 'AWARD_POINTS', payload: { teamId, points: pointsToAward } });
      }
      // Always mark as answered, even if no points awarded to a team
      dispatch({ type: 'MARK_ANSWERED', payload: { categoryId, questionId } });
    }
  };

  const wageringTeam = isWagerQuestion ? teams.find(t => t.id === wagerState.wageringTeamId) : null;
  const wagerTargetTeam = isWagerQuestion ? teams.find(t => t.id === wagerState.targetTeamId) : null;

  // Determine which team can initiate a wager (the team that just answered a NON-WAGER question)
  // This needs refinement: activeTeamIndex should track who *selected* the question.
  // For simplicity, let's assume the first team can wager if it's not a wager question.
  // A better system would track the team that picked the question.
  // For now, let's allow any team to wager if their wager is not used.
  const canTeamWager = (team) => !team.wagerUsed && !isWagerQuestion;

  return (
    <div className="flex flex-col items-center justify-center h-full p-4 md:p-8">
      <div className={`p-6 md:p-10 rounded-xl shadow-2xl w-full max-w-3xl text-center ${state.theme === 'light' ? 'bg-white' : 'bg-gray-700'}`}>
        <h2 className="text-2xl md:text-3xl font-bold mb-4">Answer:</h2>
        <p className="text-xl md:text-3xl font-semibold mb-6 text-green-500">{questionObj.answer}</p>
        <p className="text-lg md:text-xl mb-6">Question was: "{questionObj.text}" ({questionObj.points} pts)</p>

        {isWagerQuestion && wagerTargetTeam && (
          <div className="my-6">
            <h3 className="text-xl font-semibold mb-2">Wager Question for {wagerTargetTeam.name} ({wagerState.multiplier}x)</h3>
            <p className="mb-4">Did {wagerTargetTeam.name} answer correctly?</p>
            <div className="flex justify-center space-x-4">
              <Button onClick={() => handleAwardPoints(wagerTargetTeam.id, true)} className="bg-green-500 hover:bg-green-600">Correct</Button>
              <Button onClick={() => handleAwardPoints(wagerTargetTeam.id, false)} className="bg-red-500 hover:bg-red-600">Incorrect</Button>
            </div>
          </div>
        )}

        {!isWagerQuestion && (
          <div className="my-6">
            <h3 className="text-xl font-semibold mb-3">Award Points:</h3>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
              {teams.map(team => (
                <Button 
                    key={team.id} 
                    onClick={() => handleAwardPoints(team.id, true)} 
                    variant="primary"
                    className="w-full"
                >
                  Award to {team.name}
                </Button>
              ))}
            </div>
            <Button onClick={() => handleAwardPoints(null, false)} variant="secondary" className="w-full mb-6">
              No Team Answered Correctly / Continue
            </Button>

            <h3 className="text-lg font-semibold mb-2">Wager Option (Once per game per team):</h3>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
            {teams.map(team => (
                <Button 
                    key={`wager-${team.id}`}
                    onClick={() => dispatch({ type: 'INITIATE_WAGER_SETUP', payload: { wageringTeamId: team.id }})}
                    disabled={!canTeamWager(team)}
                    variant="ghost"
                    className={`border-2 ${state.theme === 'light' ? 'border-purple-400 hover:bg-purple-100' : 'border-purple-500 hover:bg-purple-800'}`}
                >
                    {team.name} Wager? {!canTeamWager(team) && '(Used or Wager Active)'}
                </Button>
            ))}
            </div>
          </div>
        )}
      </div>
    </div>
  );
};

const WagerSetupScreen = () => {
    const { state, dispatch } = useContext(GameContext);
    const wageringTeam = state.teams.find(t => t.id === state.wagerState?.wageringTeamId);
    const otherTeam = state.teams.find(t => t.id !== state.wagerState?.wageringTeamId);

    if (!wageringTeam || !otherTeam) {
        // Should not happen, navigate back
        dispatch({ type: 'NAVIGATE', payload: SCREENS.QUESTION_SELECT });
        return null;
    }

    const handleSelectTarget = () => {
        dispatch({ type: 'SET_WAGER_TARGET', payload: { targetTeamId: otherTeam.id } });
    };

    return (
        <div className="flex flex-col items-center justify-center h-full p-8">
            <h1 className="text-3xl font-bold mb-6">{wageringTeam.name} is WAGERING!</h1>
            <p className="text-xl mb-8">Choose which team will answer the wagered question:</p>
            <div className={`p-6 rounded-xl shadow-lg w-full max-w-md text-center ${state.theme === 'light' ? 'bg-white' : 'bg-gray-700'}`}>
                <Button onClick={handleSelectTarget} size="lg" className="w-full">
                    Make {otherTeam.name} Answer!
                </Button>
            </div>
            <Button onClick={() => dispatch({type: 'CLEAR_WAGER'})} variant="secondary" className="mt-8">
                Cancel Wager
            </Button>
        </div>
    );
};


const StoreScreen = () => {
  const { state, dispatch } = useContext(GameContext);
  const buyTokens = (amount) => {
    dispatch({ type: 'SET_TOKENS', payload: state.tokens + amount });
    // In a real app, this would involve a payment gateway
    dispatch({ type: 'SET_GAME_MESSAGE', payload: `Successfully bought ${amount} tokens!` });
  };

  return (
    <div className="flex flex-col items-center justify-center h-full p-8">
      <h1 className="text-4xl font-bold mb-8">Token Store</h1>
      <p className="text-xl mb-2">Your Tokens: {state.tokens}</p>
      {state.gameMessage && <p className={`mb-6 text-center font-semibold ${state.theme === 'light' ? 'text-green-600' : 'text-green-400'}`}>{state.gameMessage}</p>}
      
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6 w-full max-w-4xl">
        {[
          { amount: 10, price: '$0.99' },
          { amount: 50, price: '$3.99' },
          { amount: 100, price: '$6.99' },
        ].map(pack => (
          <div key={pack.amount} className={`p-6 rounded-xl shadow-lg text-center ${state.theme === 'light' ? 'bg-white' : 'bg-gray-700'}`}>
            <h2 className="text-3xl font-bold text-yellow-500 mb-2">{pack.amount} Tokens</h2>
            <p className="text-xl font-semibold mb-4">{pack.price}</p>
            <Button onClick={() => buyTokens(pack.amount)} size="lg" className="w-full">Buy Now</Button>
          </div>
        ))}
      </div>
      <Button onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.MAIN_MENU })} variant="secondary" size="lg" className="mt-12">
        Back to Menu
      </Button>
    </div>
  );
};

const EndGameScreen = () => {
  const { state, dispatch } = useContext(GameContext);

  const getShareableMessage = () => {
    let message = "🏆 Trivia Night Results! 🏆\n\n";
    state.teams.forEach(team => {
        message += `${team.icon} ${team.name}: ${team.score} points\n`;
        if (team.members && team.members.length > 0) {
            message += `   Members: ${team.members.join(', ')}\n`;
        }
    });
    message += "\nGreat game everyone!";
    return message;
  };

  const handleShare = () => {
    const message = getShareableMessage();
    // Basic share: copy to clipboard
    navigator.clipboard.writeText(message)
      .then(() => dispatch({ type: 'SET_GAME_MESSAGE', payload: 'Results copied to clipboard!' }))
      .catch(() => dispatch({ type: 'SET_GAME_MESSAGE', payload: 'Failed to copy results.' }));
    // In a real app, you might use Web Share API if available, or generate an image.
  };

  return (
    <div className="flex flex-col items-center justify-center h-full p-4 md:p-8">
      <h1 className="text-4xl md:text-5xl font-bold mb-6">Game Over!</h1>
      {state.gameMessage && <p className={`mb-4 text-center font-semibold ${state.theme === 'light' ? 'text-green-600' : 'text-green-400'}`}>{state.gameMessage}</p>}
      
      <div className={`p-6 md:p-8 rounded-xl shadow-2xl w-full max-w-2xl mb-8 ${state.theme === 'light' ? 'bg-white' : 'bg-gray-700'}`}>
        <h2 className="text-2xl md:text-3xl font-semibold text-center mb-6">Final Scores:</h2>
        {state.teams.sort((a,b) => b.score - a.score).map((team, index) => (
          <div key={team.id} className={`flex justify-between items-center py-3 px-4 rounded-lg mb-3 ${index === 0 ? (state.theme === 'light' ? 'bg-yellow-100' : 'bg-yellow-700') : (state.theme === 'light' ? 'bg-gray-100' : 'bg-gray-600')}`}>
            <div className="flex items-center">
              <span className="text-3xl mr-3">{team.icon}</span>
              <span className="text-lg md:text-xl font-medium">{team.name}</span>
            </div>
            <span className="text-xl md:text-2xl font-bold">{team.score} pts</span>
          </div>
        ))}
      </div>

      <div className="flex flex-col md:flex-row space-y-4 md:space-y-0 md:space-x-4 w-full max-w-2xl">
        <Button onClick={() => dispatch({ type: 'START_BONUS_CHALLENGE_SETUP' })} size="lg" className="w-full bg-purple-500 hover:bg-purple-600">
          Bonus Category Challenge?
        </Button>
        <Button onClick={handleShare} size="lg" variant="secondary" className="w-full">
          Share Results
        </Button>
      </div>
      <Button onClick={() => dispatch({ type: 'RESET_GAME' })} variant="ghost" size="lg" className="mt-8">
        Play Again (Main Menu)
      </Button>
    </div>
  );
};

const BonusChallengeSetupScreen = () => {
    const { state, dispatch } = useContext(GameContext);
    const [selectedBonusCategories, setSelectedBonusCategories] = useState([]);
    
    const unselectedCategories = MOCK_DATABASE.categories.filter(
        cat => !state.selectedCategories.includes(cat.id)
    ).slice(0, 5); // Present 5 unselected

    const handleToggleBonusCategory = (catId) => {
        setSelectedBonusCategories(prev => 
            prev.includes(catId) 
            ? prev.filter(id => id !== catId)
            : (prev.length < 2 ? [...prev, catId] : prev)
        );
    };

    const startBonus = () => {
        if (selectedBonusCategories.length === 2) {
            dispatch({ type: 'START_BONUS_ROUND', payload: selectedBonusCategories });
        }
    };

    if (unselectedCategories.length < 2) {
        return (
            <div className="flex flex-col items-center justify-center h-full p-8">
                <h1 className="text-3xl font-bold mb-4">Bonus Challenge</h1>
                <p className="text-xl mb-6">Not enough unselected categories for a bonus round.</p>
                <Button onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.END_GAME })} size="lg">Back to End Game</Button>
            </div>
        );
    }

    return (
        <div className="flex flex-col items-center h-full p-4 pt-8 md:p-8">
            <h1 className="text-3xl md:text-4xl font-bold mb-2">Bonus Category Challenge</h1>
            <p className="mb-6 text-sm md:text-base">Pick 2 categories for the bonus round. (2x points for correct, -1x for incorrect)</p>
            <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-3 lg:grid-cols-3 gap-3 md:gap-4 overflow-y-auto flex-grow w-full max-w-3xl px-2">
                {unselectedCategories.map(category => (
                    <button
                        key={category.id}
                        onClick={() => handleToggleBonusCategory(category.id)}
                        className={`relative p-3 md:p-4 rounded-2xl md:rounded-3xl aspect-[4/3] flex flex-col items-center justify-center text-center transition-all duration-200 group
                                    ${selectedBonusCategories.includes(category.id) 
                                        ? (state.theme === 'light' ? 'bg-purple-500 text-white ring-4 ring-purple-300' : 'bg-purple-600 text-white ring-4 ring-purple-400') 
                                        : (state.theme === 'light' ? 'bg-white hover:bg-gray-100 shadow-md' : 'bg-gray-700 hover:bg-gray-600 shadow-lg')}
                                    ${selectedBonusCategories.length >= 2 && !selectedBonusCategories.includes(category.id) ? 'opacity-50 cursor-not-allowed' : ''}`}
                        disabled={selectedBonusCategories.length >= 2 && !selectedBonusCategories.includes(category.id)}
                    >
                        <img src={category.image} alt={category.name} className="w-12 h-12 md:w-20 md:h-20 object-cover rounded-xl mb-2 md:mb-3 group-hover:scale-105 transition-transform" onError={(e) => e.target.src='https://placehold.co/100x100/cccccc/969696?text=Error'}/>
                        <span className="text-xs md:text-sm font-semibold">{category.name}</span>
                        {selectedBonusCategories.includes(category.id) && (
                          <div className={`absolute top-2 right-2 w-5 h-5 md:w-6 md:h-6 rounded-full flex items-center justify-center ${state.theme === 'light' ? 'bg-white text-purple-500' : 'bg-gray-800 text-purple-400'}`}>
                            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" className="w-4 h-4 md:w-5 md:h-5"><path fillRule="evenodd" d="M16.704 4.153a.75.75 0 0 1 .143 1.052l-8 10.5a.75.75 0 0 1-1.127.075l-4.5-4.5a.75.75 0 0 1 1.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 0 1 1.05-.143Z" clipRule="evenodd" /></svg>
                          </div>
                        )}
                    </button>
                ))}
            </div>
            <div className="mt-6 md:mt-8 flex space-x-4">
                <Button onClick={() => dispatch({ type: 'NAVIGATE', payload: SCREENS.END_GAME })} variant="secondary" size="lg">Decline / Back</Button>
                <Button onClick={startBonus} size="lg" disabled={selectedBonusCategories.length !== 2} className="bg-purple-500 hover:bg-purple-600">
                    Start Bonus Round
                </Button>
            </div>
        </div>
    );
};

const BonusChallengeQuestionScreen = () => {
    const { state, dispatch } = useContext(GameContext);
    const bonusRoundState = state.bonusRound;

    if (!bonusRoundState || bonusRoundState.questions.length === 0) {
        dispatch({ type: 'NAVIGATE', payload: SCREENS.END_GAME });
        return null;
    }
    
    const currentBonusQuestionData = bonusRoundState.questions[bonusRoundState.currentIndex];
    if (!currentBonusQuestionData) {
         // Should have been handled by reducer, but as a safeguard
        dispatch({ type: 'NAVIGATE', payload: SCREENS.END_GAME });
        return null;
    }

    const { teamId, category, question } = currentBonusQuestionData;
    const team = state.teams.find(t => t.id === teamId);

    const handleAnswer = (correct) => {
        dispatch({ type: 'ANSWER_BONUS_QUESTION', payload: { teamId, category, question, correct } });
    };

    return (
        <div className="flex flex-col items-center justify-center h-full p-4 md:p-8">
            <h1 className="text-3xl font-bold mb-4">Bonus Question for {team?.name}!</h1>
            <p className="text-xl mb-2">Category: {category.name}</p>
            <p className="text-lg mb-6">(Correct: +{question.points * 2} pts, Incorrect: -{question.points} pts)</p>

            <div className={`p-6 md:p-10 rounded-xl shadow-2xl w-full max-w-2xl text-center ${state.theme === 'light' ? 'bg-white' : 'bg-gray-700'}`}>
                <p className="text-xl md:text-2xl font-semibold mb-6">{question.text}</p>
                <p className="text-lg md:text-xl font-medium mb-4">Answer: <span className="text-green-500">{question.answer}</span></p>
                
                <p className="mb-4 font-semibold">Did {team?.name} answer correctly?</p>
                <div className="flex justify-center space-x-4">
                    <Button onClick={() => handleAnswer(true)} className="bg-green-500 hover:bg-green-600 px-8">Correct</Button>
                    <Button onClick={() => handleAnswer(false)} className="bg-red-500 hover:bg-red-600 px-8">Incorrect</Button>
                </div>
            </div>
        </div>
    );
};


// Main App Component
function App() {
  const [state, dispatch] = useReducer(gameReducer, initialState);

  useEffect(() => {
    // Apply theme to body for global background
    if (state.theme === 'dark') {
      document.documentElement.classList.add('dark');
      document.body.style.backgroundColor = '#1f2937'; // Tailwind gray-800
    } else {
      document.documentElement.classList.remove('dark');
      document.body.style.backgroundColor = '#fef3c7'; // Tailwind amber-100 (warmer white)
    }
  }, [state.theme]);

  const CurrentScreenComponent = () => {
    switch (state.currentScreen) {
      case SCREENS.MAIN_MENU:
        return <MainMenuScreen />;
      case SCREENS.TEAM_SETUP:
        return <TeamSetupScreen />;
      case SCREENS.CATEGORY_SELECTION:
        return <CategorySelectionScreen />;
      case SCREENS.QUESTION_SELECT:
        return <QuestionSelectScreen />;
      case SCREENS.QUESTION_DISPLAY:
        return <QuestionDisplayScreen />;
      case SCREENS.ANSWER_REVIEW:
        return <AnswerReviewScreen />;
      case SCREENS.STORE:
        return <StoreScreen />;
      case SCREENS.END_GAME:
        return <EndGameScreen />;
      case SCREENS.BONUS_CHALLENGE_SETUP:
        return <BonusChallengeSetupScreen />;
      case SCREENS.BONUS_CHALLENGE_QUESTION:
        return <BonusChallengeQuestionScreen />;
      case SCREENS.WAGER_SETUP:
        return <WagerSetupScreen />;
      default:
        return <MainMenuScreen />;
    }
  };

  return (
    <GameContext.Provider value={{ state, dispatch }}>
      <div className={`app-container min-h-screen transition-colors duration-300 ${state.theme === 'light' ? 'text-gray-900 bg-amber-50' : 'text-white bg-gray-900'} flex flex-col`}>
        <style jsx global>{`
          body {
            margin: 0;
            font-family: 'Inter', sans-serif; /* Ensure Inter is loaded or use a fallback */
            overscroll-behavior-y: contain; /* Prevents pull-to-refresh on mobile for a more app-like feel */
          }
          .app-container {
            // Force landscape-like behavior for key screens if needed, or use media queries
            // For a web app, true landscape enforcement is tricky without fullscreen API.
            // This setup focuses on responsive design that works well in wider views.
            padding-top: env(safe-area-inset-top, 0); // For notch area
            padding-bottom: env(safe-area-inset-bottom, 0);
          }
          .landscape-container { // Specific class for screens that benefit most from landscape
             // Consider min-aspect-ratio or specific width/height constraints for these
          }
          .dark {
            // Ensure all custom component styles respect dark mode
          }
          .shadow-top-md {
            box-shadow: 0 -4px 6px -1px rgba(0, 0, 0, 0.1), 0 -2px 4px -1px rgba(0, 0, 0, 0.06);
          }
        `}</style>
        <ThemeToggleButton />
        <TokenDisplay />
        <main className="flex-grow flex flex-col overflow-hidden">
          <CurrentScreenComponent />
        </main>
      </div>
    </GameContext.Provider>
  );
}

export default App;

