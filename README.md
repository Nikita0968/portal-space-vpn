[script.js](https://github.com/user-attachments/files/23576564/script.js)
// Конфигурация Supabase
const SUPABASE_URL = 'https://geigczqenuqhpvcquiyv.supabase.co';
const SUPABASE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImdlaWdjenFlbnVxaHB2Y3F1aXl2Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NjI3NzM1MTAsImV4cCI6MjA3ODM0OTUxMH0.vx2DaKrqc4aOhdo0txM__LQjcaHPf2pKutgrwzaIg5M';

// Инициализация Supabase
const supabase = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

// Telegram WebApp
let tg = window.Telegram.WebApp;
let user = null;

// Инициализация приложения
document.addEventListener('DOMContentLoaded', async function() {
    // Инициализируем Telegram WebApp
    tg.expand();
    tg.enableClosingConfirmation();
    
    // Получаем данные пользователя из Telegram
    user = tg.initDataUnsafe?.user;
    
    if (user) {
        await loadUserData(user.id);
    } else {
        // Если нет данных из Telegram, показываем ошибку
        showError('Не удалось получить данные пользователя');
    }
    
    // Скрываем лоадер
    document.getElementById('loader').classList.add('hidden');
});

// Загрузка данных пользователя
async function loadUserData(telegramId) {
    try {
        // Загружаем данные пользователя
        const { data: userData, error: userError } = await supabase
            .from('users')
            .select('*')
            .eq('telegram_id', telegramId)
            .single();

        if (userError) throw userError;

        // Обновляем интерфейс
        updateUserInterface(userData);

        // Загружаем устройства пользователя
        await loadUserDevices(userData.user_id);

    } catch (error) {
        console.error('Ошибка загрузки данных:', error);
        showError('Ошибка загрузки данных');
    }
}

// Обновление интерфейса пользователя
function updateUserInterface(userData) {
    // Основная информация
    document.getElementById('user-id').textContent = userData.telegram_id;
    document.getElementById('user-name').textContent = userData.first_name || 'Не указано';
    document.getElementById('user-username').textContent = userData.username ? `@${userData.username}` : 'Не указан';
    
    // Статистика
    const tariffElement = document.getElementById('user-tariff');
    tariffElement.textContent = getTariffName(userData.tariff);
    tariffElement.className = `value tariff ${userData.tariff}`;
    
    document.getElementById('user-balance').textContent = `${userData.balance} руб.`;
    
    const trafficLimit = userData.traffic_limit === -1 ? '∞' : `${userData.traffic_limit} MB`;
    document.getElementById('user-traffic').textContent = `${userData.traffic_used} MB / ${trafficLimit}`;
    
    // Прогресс бар трафика
    if (userData.traffic_limit !== -1) {
        const progress = (userData.traffic_used / userData.traffic_limit) * 100;
        document.getElementById('traffic-progress').style.width = `${Math.min(progress, 100)}%`;
    } else {
        document.getElementById('traffic-progress').style.width = '0%';
    }
}

// Загрузка устройств пользователя
async function loadUserDevices(userId) {
    try {
        const { data: devices, error } = await supabase
            .from('user_devices')
            .select('*')
            .eq('user_id', userId)
            .order('created_at', { ascending: false });

        if (error) throw error;

        updateDevicesInterface(devices);

    } catch (error) {
        console.error('Ошибка загрузки устройств:', error);
    }
}

// Обновление интерфейса устройств
function updateDevicesInterface(devices) {
    const devicesList = document.getElementById('devices-list');
    const devicesCount = document.getElementById('devices-count');
    
    devicesCount.textContent = devices ? devices.length : 0;
    
    if (!devices || devices.length === 0) {
        devicesList.innerHTML = `
            <div style="text-align: center; padding: 20px; color: #666;">
                <p>📱 Устройств пока нет</p>
                <p style="font-size: 12px; margin-top: 8px;">Добавьте устройство через бота</p>
            </div>
        `;
        return;
    }
    
    devicesList.innerHTML = devices.map(device => `
        <div class="device-item">
            <div class="device-info">
                <div class="device-name">${device.device_name}</div>
                <div class="device-mac">${device.mac_address}</div>
            </div>
            <div class="device-status ${device.is_active ? 'status-online' : 'status-offline'}">
                ${device.is_active ? '● Online' : '● Offline'}
            </div>
        </div>
    `).join('');
}

// Получение читаемого имени тарифа
function getTariffName(tariff) {
    const tariffNames = {
        'free': '🎯 Бесплатный',
        'standard': '🚀 Стандарт', 
        'premium': '⚡ Премиум'
    };
    return tariffNames[tariff] || tariff;
}

// Открытие бота для управления устройствами
function openBot() {
    tg.openTelegramLink('https://t.me/your_bot_username?start=devices');
}

// Открытие бота для различных действий
function openBotAction(action) {
    const actions = {
        'tariff': 'change_tariff',
        'balance': 'add_balance', 
        'settings': 'vpn_settings',
        'help': 'help'
    };
    
    const startParam = actions[action];
    if (startParam) {
        tg.openTelegramLink(`https://t.me/your_bot_username?start=${startParam}`);
    }
}

// Показать ошибку
function showError(message) {
    // Можно добавить красивый toast или уведомление
    alert(message);
}

// Обновление данных каждые 30 секунд
setInterval(() => {
    if (user) {
        loadUserData(user.id);
    }
}, 30000);

// Обработка видимости страницы
document.addEventListener('visibilitychange', function() {
    if (!document.hidden && user) {
        loadUserData(user.id);
    }
});
