<script lang="ts">
  import { onMount } from "svelte";
  import { fade, slide } from "svelte/transition";

  let isTestMode = false;

  interface Notification {
    id: string;
    type: string;
    time: string;
    remaining: number;
    createdAt: number;
    initialRemaining: number;
    icon: string;
    color: string;
  }

  let currentTime: Date = new Date();
  let notifications: Notification[] = [];

  const notificationTypes = {
    DUNGEON: {
      name: "Dungeon Spawn",
      minutes: 5,
      window: 300,
      icon: "🏰",
      color: "#4682b4",
    },
    SRANK: {
      name: "S-Rank Dungeon",
      minutes: 5,
      window: 300,
      icon: "⭐",
      color: "#e6a817",
    },
    RAID: {
      name: "Raid Start",
      minutes: 15,
      window: 300,
      icon: "⚔️",
      color: "#d35400",
    },
    MOUNT: {
      name: "Mount Spawn",
      minutes: 15,
      window: 300,
      icon: "🐎",
      color: "#2ecc71",
    },
  };

  onMount(() => {
    console.log("Component mounted, starting interval");
    
    const urlParams = new URLSearchParams(window.location.search);
    isTestMode = urlParams.get('test') === 'true';
    
    if (isTestMode) {
      console.log("Test mode activated, adding sample notifications");
      addSampleNotifications();
    }
    
    const interval = setInterval(() => {
      currentTime = new Date();
      updateNotifications();
    }, 1000);
    
    return () => {
      console.log("Component unmounted, clearing interval");
      clearInterval(interval);
    };
  });

  function addSampleNotifications() {
    const now = new Date();
    const currentHour = now.getHours();
    const currentMinute = now.getMinutes();
    
    const sampleNotifications = [
      createNotification(
        notificationTypes.DUNGEON.name,
        `${currentHour.toString().padStart(2, "0")}:${(currentMinute + 5).toString().padStart(2, "0")}`,
        300
      ),
      createNotification(
        notificationTypes.RAID.name,
        `${currentHour.toString().padStart(2, "0")}:${(currentMinute + 1).toString().padStart(2, "0")}`,
        60
      ),
      createNotification(
        notificationTypes.SRANK.name,
        `${currentHour.toString().padStart(2, "0")}:${currentMinute.toString().padStart(2, "0")}`,
        30
      ),
      createNotification(
        notificationTypes.MOUNT.name,
        `${currentHour.toString().padStart(2, "0")}:${currentMinute.toString().padStart(2, "0")}`,
        10
      ),
      createNotification(
        notificationTypes.DUNGEON.name,
        `${currentHour.toString().padStart(2, "0")}:${currentMinute.toString().padStart(2, "0")}`,
        0
      )
    ];
    
    notifications = [...notifications, ...sampleNotifications];
  }

  const dungeonSpawnTimes: string[] = ["00", "30"];
  const sRankTimes: string[] = [
    "13:00",
    "16:00",
    "19:00",
    "22:00",
    "01:00",
    "04:00",
    "07:00",
  ];
  const raidStartTimes: string[] = ["15", "45"];
  const mountSpawnTimes: string[] = ["15", "30", "45", "58"];

  function getTimeRemaining(targetHour: number, targetMinute: number): number {
    const now = new Date();
    let target = new Date(now);
    target.setHours(targetHour, targetMinute, 0, 0);
    if (target < now) target.setDate(target.getDate() + 1);
    const remaining = Math.max(
      0,
      Math.floor((target.getTime() - now.getTime()) / 1000)
    );
    return remaining;
  }

  function formatCountdown(seconds: number): string {
    const mins = Math.floor(seconds / 60)
      .toString()
      .padStart(2, "0");
    const secs = (seconds % 60).toString().padStart(2, "0");
    return `${mins}:${secs}`;
  }

  function createNotification(
    type: string,
    time: string,
    remaining: number
  ): Notification {
    const typeConfig = Object.values(notificationTypes).find(
      (t) => t.name === type
    );
    const icon = typeConfig?.icon || "🔔";
    const color = typeConfig?.color || "#4682b4";

    return {
      id: `${type}-${time}-${Date.now()}`,
      type,
      time,
      remaining,
      initialRemaining: remaining,
      createdAt: Date.now(),
      icon,
      color,
    };
  }

  function checkEvents(): Notification[] {
    if (isTestMode) {
      return [];
    }
    
    const now = new Date();
    const newNotifications: Notification[] = [];

    dungeonSpawnTimes.forEach((minute) => {
      const timeLeft = getTimeRemaining(now.getHours(), parseInt(minute));
      if (timeLeft <= 300 || (timeLeft >= 86340 && timeLeft < 86400)) {
        const event = createNotification(
          notificationTypes.DUNGEON.name,
          `${now.getHours().toString().padStart(2, "0")}:${minute}`,
          timeLeft > 86000 ? 0 : timeLeft
        );
        newNotifications.push(event);
      }
    });

    sRankTimes.forEach((time) => {
      const [hour, minute] = time.split(":").map(Number);
      const timeLeft = getTimeRemaining(hour, minute);
      if (timeLeft <= 300 || (timeLeft >= 86340 && timeLeft < 86400)) {
        const event = createNotification(
          notificationTypes.SRANK.name,
          time,
          timeLeft > 86000 ? 0 : timeLeft
        );
        newNotifications.push(event);
      }
    });

    raidStartTimes.forEach((minute) => {
      const timeLeft = getTimeRemaining(now.getHours(), parseInt(minute));
      if (timeLeft <= 300 || (timeLeft >= 86340 && timeLeft < 86400)) {
        const event = createNotification(
          notificationTypes.RAID.name,
          `${now.getHours().toString().padStart(2, "0")}:${minute}`,
          timeLeft > 86000 ? 0 : timeLeft
        );
        newNotifications.push(event);
      }
    });

    mountSpawnTimes.forEach((minute) => {
      const timeLeft = getTimeRemaining(now.getHours(), parseInt(minute));
      if (timeLeft <= 300 || (timeLeft >= 86340 && timeLeft < 86400)) {
        const event = createNotification(
          notificationTypes.MOUNT.name,
          `${now.getHours().toString().padStart(2, "0")}:${minute}`,
          timeLeft > 86000 ? 0 : timeLeft
        );
        newNotifications.push(event);
      }
    });

    return newNotifications;
  }

  function updateNotifications(): void {
    if (!isTestMode) {
      const newNotifications = checkEvents();
      if (newNotifications.length > 0) {
        const filteredNewNotifications = newNotifications.filter((newNotif) => {
          return !notifications.some(
            (existing) =>
              existing.type === newNotif.type && existing.time === newNotif.time
          );
        });
        if (filteredNewNotifications.length > 0) {
          notifications = [...notifications, ...filteredNewNotifications];
        }
      }
    }

    notifications = notifications.map((notification) => {
      if (isTestMode) {
        return {
          ...notification,
          remaining: Math.max(0, notification.remaining - 1)
        };
      }
      
      let targetHour, targetMinute;
      if (notification.type === notificationTypes.SRANK.name) {
        [targetHour, targetMinute] = notification.time.split(":").map(Number);
      } else {
        targetHour = parseInt(notification.time.split(":")[0]);
        targetMinute = parseInt(notification.time.split(":")[1]);
      }

      const timeLeft = getTimeRemaining(targetHour, targetMinute);
      return {
        ...notification,
        remaining: timeLeft > 86000 ? 0 : timeLeft,
      };
    });

    notifications = notifications.filter((n) => {
      const typeConfig = Object.values(notificationTypes).find(
        (t) => t.name === n.type
      );
      const windowSeconds = typeConfig?.window || 300;

      if (isTestMode) {
        if (n.remaining > 0) return true;
        const timeSinceZero = (Date.now() - n.createdAt) / 1000 - n.initialRemaining;
        return timeSinceZero < windowSeconds;
      }

      let targetHour, targetMinute;
      if (n.type === notificationTypes.SRANK.name) {
        [targetHour, targetMinute] = n.time.split(":").map(Number);
      } else {
        targetHour = parseInt(n.time.split(":")[0]);
        targetMinute = parseInt(n.time.split(":")[1]);
      }

      const timeLeft = getTimeRemaining(targetHour, targetMinute);
      const secondsSinceEvent = timeLeft > 86000 ? (86400 - timeLeft) : -timeLeft;

      return (
        timeLeft > 0 ||
        (timeLeft >= 86340 && timeLeft < 86400) ||
        secondsSinceEvent < windowSeconds
      );
    });
  }

  function isUrgent(remaining: number): boolean {
    return remaining <= 60 && remaining > 0;
  }

  function isVeryUrgent(remaining: number): boolean {
    return remaining <= 30 && remaining > 0;
  }
</script>

<div class="app-wrapper">
  <div class="toast-container">
    {#each notifications as notification (notification.id)}
      <div
        class="toast"
        class:urgent={isUrgent(notification.remaining)}
        class:very-urgent={isVeryUrgent(notification.remaining)}
        in:slide={{ duration: 400, delay: 100 }}
        out:fade={{ duration: 400 }}
        style="border-left-color: {notification.color}"
      >
        <div class="notification-content">
          <span class="icon">{notification.icon || "🔔"}</span>
          <div class="notification-text">
            <span
              class:highlight={notification.remaining <= 60}
              style="color: {notification.remaining <= 60
                ? '#ff7f50'
                : '#000000'}"
            >
              {notification.type} at {notification.time}
            </span>
          </div>
          <span
            class="countdown"
            class:urgent={isUrgent(notification.remaining)}
            class:very-urgent={isVeryUrgent(notification.remaining)}
            style="color: {!isUrgent(notification.remaining) &&
            !isVeryUrgent(notification.remaining)
              ? notification.color
              : ''}"
          >
            {notification.remaining === 0
              ? "Now!"
              : formatCountdown(notification.remaining)}
          </span>
        </div>
      </div>
    {/each}
  </div>
</div>

<style>
  @import url("https://fonts.googleapis.com/css2?family=Prompt:ital,wght@0,100;0,200;0,300;0,400;0,500;0,600;0,700;0,800;0,900;1,100;1,200;1,300;1,400;1,500;1,600;1,700;1,800;1,900&display=swap");

  :global(*) {
    font-family: "Prompt", sans-serif !important;
  }

  .app-wrapper {
    width: 100%;
    height: 100%;
    position: fixed;
    top: 0;
    left: 0;
    background-color: transparent;
    z-index: 999;
  }

  .toast-container {
    position: fixed;
    bottom: 20px;
    right: 20px;
    width: 300px;
    z-index: 1000;
  }
  
  .toast {
    background: rgba(255, 255, 255, 0.95);
    color: #000000;
    padding: 12px 16px;
    margin: 8px 0;
    border-radius: 8px;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.3);
    border-left: 4px solid;
    font-size: 14px;
    position: relative;
    overflow: hidden;
    transition: all 0.3s ease;
    backdrop-filter: blur(5px);
  }

  .toast.urgent {
    background-color: rgba(255, 248, 225, 0.95);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
    transform: scale(1.05);
  }

  .toast.very-urgent {
    background-color: rgba(255, 235, 238, 0.95);
    box-shadow: 0 4px 12px rgba(255, 0, 0, 0.4);
    transform: scale(1.1);
    animation: pulse 1.5s infinite;
  }

  @keyframes pulse {
    0% { transform: scale(1.05); }
    50% { transform: scale(1.1); }
    100% { transform: scale(1.05); }
  }

  .notification-content {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  .notification-text {
    display: flex;
    align-items: center;
    gap: 8px;
    flex: 1;
  }
  .icon {
    font-size: 16px;
    margin-right: 4px;
  }
  .highlight {
    font-weight: 600;
  }
  .countdown {
    margin-left: 8px;
    font-size: 12px;
    white-space: nowrap;
  }

  .countdown.urgent {
    font-weight: bold;
    font-size: 14px;
    color: #ff7f50 !important;
  }

  .countdown.very-urgent {
    font-weight: bold;
    font-size: 16px;
    color: #ff0000 !important;
    animation: blink 1s infinite;
  }

  @keyframes blink {
    0% { opacity: 1; }
    50% { opacity: 0.5; }
    100% { opacity: 1; }
  }
</style>
