const TelegramBot = require('node-telegram-bot-api');
const fs = require('fs');


// Замените 'YOUR_TELEGRAM_BOT_TOKEN' на ваш токен бота
const token = '7605139245:AAGd5hh0NRlnexNwTOBaTdBcdAR0MV-e-iU';
const bot = new TelegramBot(token, { polling: true });


// Загрузка данных из JSON файла
function loadResponses() {
    if (!fs.existsSync('responses.json')) {
        fs.writeFileSync('responses.json', JSON.stringify({}, null, 4), 'utf-8');
    }
    const data = fs.readFileSync('responses.json', 'utf-8');
    return JSON.parse(data);
}


const responses = loadResponses();


// Инициализация состояния игры
const currentGame = {};


// Функция для установки статуса игры
function setInGameStatus(chatId, status) {
    currentGame[chatId] = status;
}


// Функция для проверки статуса игры
function isInGame(chatId) {
    return currentGame[chatId] && currentGame[chatId].type;
}


// Функция для обработки сообщений во время игры
function handleGameMessage(chatId, messageText) {
    if (messageText === 'прекратить игру') {
        bot.sendMessage(chatId, 
            "<i>Игра завершена. Вы можете начать новую игру или продолжить диалог.</i>", 
            { parse_mode: 'HTML' }
        );
        setInGameStatus(chatId, false);
        return true; // Выход из игры
    }
    return false; // Продолжение игры
}


// Функция для начала игры "Угадай число"
function startNumberGame(chatId) {
    const randomNumber = Math.floor(Math.random() * 100) + 1;
    bot.sendMessage(chatId, 
        "<b>Давай сыграем в игру 'Угадай число' 🎮!</b>\n" +
        "Я загадал число от 1 до 100. У тебя есть 10 попыток, чтобы угадать его.", 
        { parse_mode: 'HTML' }
    );


    setInGameStatus(chatId, { type: 'number', number: randomNumber, attempts: 10 });


    bot.sendMessage(chatId, 
        "<i>Выберите уровень сложности: легкий, средний или сложный.</i>", 
        { parse_mode: 'HTML' }
    );


    bot.once('message', (msg) => {
        handleNumberGameDifficulty(chatId, msg);
    });
}


// Функция для обработки выбора уровня сложности
function handleNumberGameDifficulty(chatId, msg) {
    const difficulty = msg.text.toLowerCase().trim();
    if (difficulty === 'легкий') {
        currentGame[chatId].attempts = 15;
        currentGame[chatId].difficulty = 'easy';
    } else if (difficulty === 'средний') {
        currentGame[chatId].attempts = 10;
        currentGame[chatId].difficulty = 'medium';
    } else if (difficulty === 'сложный') {
        currentGame[chatId].attempts = 5;
        currentGame[chatId].difficulty = 'hard';
    } else {
        bot.sendMessage(chatId, 
            "<i>Извините, я не понял ваш выбор. Пожалуйста, выберите уровень сложности: легкий, средний или сложный.</i>", 
            { parse_mode: 'HTML' }
        );
        bot.once('message', (msg) => {
            handleNumberGameDifficulty(chatId, msg);
        });
        return;
    }


    bot.sendMessage(chatId, 
        "<i>Вы выбрали уровень сложности: " + currentGame[chatId].difficulty + ". У вас есть " + currentGame[chatId].attempts + " попыток.</i>", 
        { parse_mode: 'HTML' }
    );


    bot.once('message', (msg) => {
        handleNumberGameMessage(chatId, msg);
    });
}


// Функция для обработки сообщений во время игры "Угадай число"
function handleNumberGameMessage(chatId, msg) {
    const userGuess = parseInt(msg.text, 10);
    if (isNaN(userGuess)) {
        bot.sendMessage(chatId, "<i>Пожалуйста, введите число! 🧐</i>", { parse_mode: 'HTML' });
        bot.once('message', (msg) => {
            handleNumberGameMessage(chatId, msg);
        });
        return;
    }


    if (handleGameMessage(chatId, msg.text.toLowerCase().trim())) {
        return;
    }


    const attemptsLeft = currentGame[chatId].attempts - 1;


    if (userGuess === currentGame[chatId].number) {
        bot.sendMessage(chatId, 
            `<b>Поздравляю! 🎉</b> Ты угадал число: <code>${currentGame[chatId].number}</code> с ${currentGame[chatId].attempts - attemptsLeft} попытки.`, 
            { parse_mode: 'HTML' }
        );
        bot.sendMessage(chatId, 
            "<i>Хотите сыграть еще раз? Напишите 'начать игру число' или 'прекратить игру'.</i>", 
            { parse_mode: 'HTML' }
        );
        setInGameStatus(chatId, false);
    } else if (userGuess < currentGame[chatId].number) {
        bot.sendMessage(chatId, 
            `<i>Твое число меньше загаданного. Осталось ${attemptsLeft} попыток. Попробуй еще раз! 🧐</i>`, 
            { parse_mode: 'HTML' }
        );
    } else {
        bot.sendMessage(chatId, 
            `<i>Твое число больше загаданного. Осталось ${attemptsLeft} попыток. Попробуй еще раз! 🧐</i>`, 
            { parse_mode: 'HTML' }
        );
    }


    if (attemptsLeft > 0) {
        currentGame[chatId].attempts = attemptsLeft;
        bot.once('message', (msg) => {
            handleNumberGameMessage(chatId, msg);
        });
    } else {
        bot.sendMessage(chatId, 
            `<i>К сожалению, ты не угадал число: <code>${currentGame[chatId].number}</code>. Игра окончена.</i>`, 
            { parse_mode: 'HTML' }
        );
        setInGameStatus(chatId, false);
    }
}


// Функция для начала игры "Угадай слово"
function startWordGame(chatId) {
    const words = ['машина', 'компьютер', 'телефон', 'космос', 'программирование', 'велосипед', 'книга', 'солнце', 'друг', 'город'];
    const currentWord = words[Math.floor(Math.random() * words.length)];
    const hiddenWord = currentWord.replace(/[а-яё]/gi, '_');
    bot.sendMessage(chatId, 
        "<b>Давай сыграем в игру 'Угадай слово'! 🧠</b>\n" +
        "Я загадал слово, но я скрываю все буквы, попробуй угадать.\n" +
        `Вот слово: <code>${hiddenWord}</code>`, 
        { parse_mode: 'HTML' }
    );


    setInGameStatus(chatId, { type: 'word', word: currentWord, attempts: 5 });


    bot.sendMessage(chatId, 
        "<i>Выберите уровень сложности: легкий, средний или сложный.</i>", 
        { parse_mode: 'HTML' }
    );


    bot.once('message', (msg) => {
        handleWordGameDifficulty(chatId, msg);
    });
}


// Функция для обработки выбора уровня сложности
function handleWordGameDifficulty(chatId, msg) {
    const difficulty = msg.text.toLowerCase().trim();
    if (difficulty === 'легкий') {
        currentGame[chatId].attempts = 10;
        currentGame[chatId].difficulty = 'easy';
    } else if (difficulty === 'средний') {
        currentGame[chatId].attempts = 5;
        currentGame[chatId].difficulty = 'medium';
    } else if (difficulty === 'сложный') {
        currentGame[chatId].attempts = 3;
        currentGame[chatId].difficulty = 'hard';
    } else {
        bot.sendMessage(chatId, 
            "<i>Извините, я не понял ваш выбор. Пожалуйста, выберите уровень сложности: легкий, средний или сложный.</i>", 
            { parse_mode: 'HTML' }
        );
        bot.once('message', (msg) => {
            handleWordGameDifficulty(chatId, msg);
        });
        return;
    }


    bot.sendMessage(chatId, 
        "<i>Вы выбрали уровень сложности: " + currentGame[chatId].difficulty + ". У вас есть " + currentGame[chatId].attempts + " попыток.</i>", 
        { parse_mode: 'HTML' }
    );


    bot.once('message', (msg) => {
        handleWordGameMessage(chatId, msg);
    });
}


// Функция для обработки сообщений во время игры "Угадай слово"
function handleWordGameMessage(chatId, msg) {
    const userGuess = msg.text.toLowerCase().trim();


    if (handleGameMessage(chatId, msg.text.toLowerCase().trim())) {
        return;
    }


    const attemptsLeft = currentGame[chatId].attempts - 1;


    if (userGuess === currentGame[chatId].word) {
        bot.sendMessage(chatId, 
            `<b>Поздравляю! 🎉</b> Ты угадал слово: <code>${currentGame[chatId].word}</code> с ${currentGame[chatId].attempts - attemptsLeft} попытки.`, 
            { parse_mode: 'HTML' }
        );
        bot.sendMessage(chatId, 
            "<i>Хотите сыграть еще раз? Напишите 'начать игру слово' или 'прекратить игру'.</i>", 
            { parse_mode: 'HTML' }
        );
        setInGameStatus(chatId, false);
    } else {
        if (attemptsLeft > 0) {
            bot.sendMessage(chatId, 
                `<i>Неправильно! Осталось ${attemptsLeft} попыток. Попробуй еще раз! 🧐</i>`, 
                { parse_mode: 'HTML' }
            );
        } else {
            bot.sendMessage(chatId, 
                `<i>К сожалению, ты не угадал слово: <code>${currentGame[chatId].word}</code>. Игра окончена.</i>`, 
                { parse_mode: 'HTML' }
            );
            setInGameStatus(chatId, false);
        }
    }


    if (attemptsLeft > 0) {
        currentGame[chatId].attempts = attemptsLeft;
        bot.once('message', (msg) => {
            handleWordGameMessage(chatId, msg);
        });
    }
}


// Обработчик команды /start
bot.onText(/\/start/, (msg) => {
    const chatId = msg.chat.id;
    bot.sendMessage(chatId, 
        "<b>Привет!</b> Я ваш друг Рома 😊\n\n" +
        "Напишите мне что-нибудь, и я постараюсь ответить! \n" +
        "Вы можете начать игру, написав 'начать игру число' или 'начать игру слово'.\n" +
        "Также вы можете просто пообщаться со мной.", 
        { parse_mode: 'HTML' }
    );
});


// Обработчик команды /help
bot.onText(/\/help/, (msg) => {
    const chatId = msg.chat.id;
    bot.sendMessage(chatId, 
        "<i>Я могу ответить на ваши вопросы или просто поболтать.</i>\n" +
        "Чтобы начать игру, напишите 'начать игру число' или 'начать игру слово'.\n" +
        "Также вы можете использовать команду <code>/start</code> для начала.", 
        { parse_mode: 'HTML' }
    );
});


// Обработчик сообщений
bot.on('message', (msg) => {
    const chatId = msg.chat.id;
    const messageText = msg.text.toLowerCase().trim();


    // Проверка, находится ли пользователь в игре
    if (isInGame(chatId)) {
        const isGameEnded = handleGameMessage(chatId, messageText);
        if (isGameEnded) {
            return;
        }
        // Если игра не завершена, продолжаем игру
        if (currentGame[chatId].type === 'number') {
            handleNumberGameMessage(chatId, msg);
        } else if (currentGame[chatId].type === 'word') {
            handleWordGameMessage(chatId, msg);
        }
        return;
    }


    // Обработка команды "начать игру"
    if (messageText.startsWith('начать игру')) {
        const gameType = messageText.split(' ').pop();
        if (gameType === 'число') {
            startNumberGame(chatId);
        } else if (gameType === 'слово') {
            startWordGame(chatId);
        } else {
            bot.sendMessage(chatId, 
                "<i>Извините, я не понял, какую игру вы хотите начать. Пожалуйста, уточните.</i>", 
                { parse_mode: 'HTML' }
            );
        }
        return;
    }


    // Обработка приветствий
    else if (messageText.includes('привет') || messageText.includes('здравствуй') || messageText.includes('хай') || messageText.includes('добрый день') || messageText.includes('добрый вечер') || messageText.includes('доброе утро')) {
        const response = getRandomResponse('greetings');
        bot.sendMessage(chatId, 
            `<b>${response}</b>`, 
            { parse_mode: 'HTML' }
        );
    }


    // Обработка вопроса "как дела"
    else if (messageText.includes('как дела') || messageText.includes('как твои дела') || messageText.includes('как ты') || messageText.includes('как поживаешь') || messageText.includes('как поживаете')) {
        const response = getRandomResponse('how_are_you');
        bot.sendMessage(chatId, 
            `<i>${response}</i>`, 
            { parse_mode: 'HTML' }
        );
    }


    // Обработка интереса к погоде
    else if (messageText.includes('погода') || messageText.includes('какая погода') || messageText.includes('прогноз погоды') || messageText.includes('как погода')) {
        const response = getRandomResponse('weather');
        bot.sendMessage(chatId, 
            `<i>${response}</i>`, 
            { parse_mode: 'HTML' }
        );
    }


    // Обработка реакции на смешные сообщения
    else if (messageText.includes('смешно') || messageText.includes('анекдот') || messageText.includes('шутка') || messageText.includes('юмор')) {
        const response = getRandomResponse('funny');
        bot.sendMessage(chatId, 
            `<b>${response}</b>`, 
            { parse_mode: 'HTML' }
        );
    }


    // Обработка диалоговых команд
    else if (messageText.includes('расскажи анекдот') || messageText.includes('расскажи шутку') || messageText.includes('расскажи анекдоты') || messageText.includes('расскажи шутки')) {
        const response = getRandomResponse('dialogue', 'tell_joke');
        bot.sendMessage(chatId, 
            `<i>${response}</i>`, 
            { parse_mode: 'HTML' }
        );
    } else if (messageText.includes('расскажи что-нибудь интересное') || messageText.includes('интересный факт') || messageText.includes('расскажи интересный факт') || messageText.includes('расскажи интересные факты')) {
        const response = getRandomResponse('dialogue', 'tell_interesting');
        bot.sendMessage(chatId, 
            `<i>${response}</i>`, 
            { parse_mode: 'HTML' }
        );
    } else if (messageText.includes('расскажи факт') || messageText.includes('факт') || messageText.includes('расскажи факты')) {
        const response = getRandomResponse('dialogue', 'tell_fact');
        bot.sendMessage(chatId, 
            `<i>${response}</i>`, 
            { parse_mode: 'HTML' }
        );
    } else if (messageText.includes('расскажи историю') || messageText.includes('история') || messageText.includes('расскажи истории') || messageText.includes('расскажи историю') || messageText.includes('расскажи историю') || messageText.includes('расскажи историю') || messageText.includes('расскажи историю') || messageText.includes('расскажи историю') || messageText.includes('расскажи историю')) {
        const response = getRandomResponse('dialogue', 'tell_story');
        bot.sendMessage(chatId, 
            `<i>${response}</i>`, 
            { parse_mode: 'HTML' }
        );
    } else if (messageText.includes('расскажи сказку') || messageText.includes('сказка') || messageText.includes('расскажи сказки') || messageText.includes('расскажи сказку') || messageText.includes('расскажи сказки')) {
        const response = getRandomResponse('dialogue', 'tell_fairy_tale');
        bot.sendMessage(chatId, 
            `<i>${response}</i>`, 
            { parse_mode: 'HTML' }
        );
    }


    // Обработка нераспознанных сообщений
    else {
        bot.sendMessage(chatId, 
            "<i>Я не совсем понял, о чем вы говорите. Попробуйте еще раз!</i>", 
            { parse_mode: 'HTML' }
        );
    }
});


// Функция для получения случайного ответа
function getRandomResponse(intent, subdialogue = null) {
    if (intent === 'dialogue' && subdialogue) {
        const subdialogueData = responses.dialogue.subdialogues[subdialogue];
        if (subdialogueData) {
            return randomElement(subdialogueData.responses);
        }
    } else {
        const intentData = responses[intent];
        if (intentData) {
            return randomElement(intentData.responses);
        }
    }
    return "Извините, я не могу найти подходящий ответ.";
}


// Функция для получения случайного элемента из массива
function randomElement(array) {
    return array[Math.floor(Math.random() * array.length)];
}


// Запуск бота
bot.on('polling_error', (error) => {
    console.log(error);
});
