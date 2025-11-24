const express = require('express');
const axios = require('axios');
const rateLimit = require('express-rate-limit');
const cors = require('cors');
// Використовуємо dotenv для локального тестування. На Render ключ буде у змінних оточення.
require('dotenv').config(); 

const app = express();

// ВИПРАВЛЕНО: Використовуємо порт від Render (process.env.PORT) або 3000 для локального тестування.
const PORT = process.env.PORT || 3000; 

// Ваш ключ API буде автоматично взятий з Render (змінні оточення)
const API_KEY = process.env.OPENWEATHERMAP_API_KEY;

// === СПИСОК ВАЛІДНИХ МІСТ ДЛЯ ШВЕЦІЇ ===
const VALID_CITIES = [
    'Стокгольм', 'Гетеборг', 'Мальме', 'Уппсала', 'Вестерос', 
    'Еребру', 'Лінчепінг', 'Гельсінборг', 'Норчепінг', 'Євле'
];

// 1. Налаштування CORS
const corsOptions = {
    // Дозволяємо запити з будь-якого джерела (для тестування). Пізніше замінити на домен Front-end.
    origin: '*', 
    methods: 'GET,POST',
};
app.use(cors(corsOptions));
app.use(express.json());

// 2. Локальне (IP-базоване) Лімітування (60 запитів за 15 хвилин на один IP)
const userLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, 
    max: 60, 
    message: JSON.stringify({
        error: 'Забагато запитів. Спробуйте пізніше через 15 хвилин.',
        status: 429
    }),
    standardHeaders: true,
    legacyHeaders: false,
});
app.use('/api/', userLimiter);

// 3. Глобальне Лімітування (Захист від перевищення добового ліміту OpenWeatherMap)
const GLOBAL_DAILY_LIMIT = 900; 
let dailyRequestCount = 0; 
let lastResetDate = new Date().getDate(); 

const checkGlobalLimit = (req, res, next) => {
    const today = new Date().getDate();

    if (today !== lastResetDate) {
        dailyRequestCount = 0;
        lastResetDate = today;
    }

    if (dailyRequestCount >= GLOBAL_DAILY_LIMIT) {
        return res.status(429).json({
            error: 'Вичерпано добовий ліміт API. Спробуйте завтра.',
            status: 429
        });
    }

    next();
};
app.use('/api/', checkGlobalLimit);

// === ФУНКЦІЯ ОБРОБКИ ДАНИХ ДЛЯ FRONT-END ===
function processWeatherData(owmData) {
    if (!owmData || !Array.isArray(owmData.list)) {
        return [];
    }
    // Вибираємо перші 8 точок (24 години) 
    const forecastList = owmData.list.slice(0, 8); 

    return forecastList.map(dataPoint => {
        return {
            temp: dataPoint.main.temp,
            feelsLike: dataPoint.main.feels_like,
            humidity: dataPoint.main.humidity,
            windSpeed: dataPoint.wind.speed,
            clouds: dataPoint.clouds.all, 
            mainWeather: dataPoint.weather[0].main,
            description: dataPoint.weather[0].description,
        };
    });
}

// 4. Основний маршрут проксі-сервера
app.post('/api/weather', async (req, res) => {
    const { city } = req.body;

    // ПЕРЕВІРКА №1: Наявність API ключа
    if (!API_KEY) {
        return res.status(500).json({ error: 'API Key not configured on server.', status: 500 });
    }

    // ПЕРЕВІРКА №2: Валідація міста на стороні сервера (Захист від зайвих запитів до OWM)
    if (!city || !VALID_CITIES.includes(city)) {
        return res.status(400).json({ 
            error: 'Невалідний запит. Оберіть місто зі списку дозволених міст Швеції.', 
            status: 400 
        });
    }

    // Збільшуємо глобальний лічильник тільки після того, як місто пройшло валідацію
    dailyRequestCount++; 

    try {
        // ВИПРАВЛЕНО: Додаємо ,SE для гарантованого пошуку Швеції
        const forecastUrl = `https://api.openweathermap.org/data/2.5/forecast?q=${city},SE&appid=${API_KEY}&units=metric&lang=uk`;
        const response = await axios.get(forecastUrl);
        
        const processedForecast = processWeatherData(response.data);

        res.json({
            city: response.data.city.name,
            forecast: processedForecast
        });
    
    } catch (error) {
        const status = error.response ? error.response.status : 500;
        const message = error.response && error.response.data.message
            ? `OpenWeatherMap: ${error.response.data.message}`
            : 'Помилка отримання даних про погоду';

        res.status(status).json({ error: message, status: status });
    }
});

// Запуск сервера
app.listen(PORT, () => {
    // ВИПРАВЛЕНО: Повідомлення про запуск без локальної IP-адреси
    console.log(`Proxy server running on port ${PORT}`);
});
