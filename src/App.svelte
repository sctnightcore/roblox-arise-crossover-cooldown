<script lang="ts">
  import { onMount } from "svelte";
  import { fade, slide, fly } from "svelte/transition";

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
      window: 2,
      icon: "🏰",
      color: "#4682b4",
    },
    SRANK: {
      name: "S-Rank Dungeon",
      minutes: 5,
      window: 2,
      icon: "⭐",
      color: "#e6a817",
    },
    RAID: {
      name: "Raid Start",
      minutes: 15,
      window: 2,
      icon: "⚔️",
      color: "#d35400",
    },
    MOUNT: {
      name: "Mount Spawn",
      minutes: 15,
      window: 2,
      icon: "🐎",
      color: "#2ecc71",
    },
  };

  onMount(() => {
    console.log("Component mounted, starting interval");
    const interval = setInterval(() => {
      currentTime = new Date();
      updateNotifications();
    }, 1000);
    return () => {
      console.log("Component unmounted, clearing interval");
      clearInterval(interval);
    };
  });

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
  const mountSpawnTimes: string[] = ["15", "30", "45"];

  function getTimeRemaining(targetHour: number, targetMinute: number): number {
    const now = new Date();
    let target = new Date(now);
    target.setHours(targetHour, targetMinute, 0, 0);
    if (target < now) target.setDate(target.getDate() + 1);
    return Math.max(0, Math.floor((target.getTime() - now.getTime()) / 1000));
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
    const now = new Date();
    const newNotifications: Notification[] = [];
    dungeonSpawnTimes.forEach((minute) => {
      const timeLeft = getTimeRemaining(now.getHours(), parseInt(minute));
      if (timeLeft <= 300 || (timeLeft >= 86340 && timeLeft < 86400)) {
        newNotifications.push(
          createNotification(
            notificationTypes.DUNGEON.name,
            `${now.getHours().toString().padStart(2, "0")}:${minute}`,
            timeLeft > 86000 ? 0 : timeLeft
          )
        );
      }
    });
    sRankTimes.forEach((time) => {
      const [hour, minute] = time.split(":").map(Number);
      const timeLeft = getTimeRemaining(hour, minute);
      if (timeLeft <= 300 || (timeLeft >= 86340 && timeLeft < 86400)) {
        newNotifications.push(
          createNotification(
            notificationTypes.SRANK.name,
            time,
            timeLeft > 86000 ? 0 : timeLeft
          )
        );
      }
    });
    raidStartTimes.forEach((minute) => {
      const timeLeft = getTimeRemaining(now.getHours(), parseInt(minute));
      if (timeLeft <= 300 || (timeLeft >= 86340 && timeLeft < 86400)) {
        newNotifications.push(
          createNotification(
            notificationTypes.RAID.name,
            `${now.getHours().toString().padStart(2, "0")}:${minute}`,
            timeLeft > 86000 ? 0 : timeLeft
          )
        );
      }
    });
    mountSpawnTimes.forEach((minute) => {
      const timeLeft = getTimeRemaining(now.getHours(), parseInt(minute));
      if (timeLeft <= 300 || (timeLeft >= 86340 && timeLeft < 86400)) {
        newNotifications.push(
          createNotification(
            notificationTypes.MOUNT.name,
            `${now.getHours().toString().padStart(2, "0")}:${minute}`,
            timeLeft > 86000 ? 0 : timeLeft
          )
        );
      }
    });
    return newNotifications;
  }

  function updateNotifications(): void {
    const newNotifications = checkEvents();
    console.log("New notifications from checkEvents:", newNotifications);

    if (newNotifications.length > 0) {
      const filteredNewNotifications = newNotifications.filter((newNotif) => {
        return !notifications.some(
          (existing) =>
            existing.type === newNotif.type && existing.time === newNotif.time
        );
      });
      console.log(
        "Filtered new notifications (after removing duplicates):",
        filteredNewNotifications
      );

      if (filteredNewNotifications.length > 0) {
        notifications = [...notifications, ...filteredNewNotifications];
        console.log(
          "Updated notifications array after adding new ones:",
          notifications
        );
      }
    }

    notifications = notifications.map((notification) => {
      let targetHour, targetMinute;
      if (notification.type === notificationTypes.SRANK.name) {
        [targetHour, targetMinute] = notification.time.split(":").map(Number);
      } else {
        [targetHour, targetMinute] = [
          parseInt(notification.time.split(":")[0]),
          parseInt(notification.time.split(":")[1]),
        ];
      }
      const timeLeft = getTimeRemaining(targetHour, targetMinute);
      console.log(
        `Updating ${notification.type} at ${notification.time}: timeLeft=${timeLeft}`
      );
      return { ...notification, remaining: timeLeft > 86000 ? 0 : timeLeft };
    });

    notifications = notifications.filter((n) => {
      const typeConfig = Object.values(notificationTypes).find(
        (t) => t.name === n.type
      );
      const windowSeconds = (typeConfig?.window || 1) * 60;

      let targetHour, targetMinute;
      if (n.type === notificationTypes.SRANK.name) {
        [targetHour, targetMinute] = n.time.split(":").map(Number);
      } else {
        [targetHour, targetMinute] = [
          parseInt(n.time.split(":")[0]),
          parseInt(n.time.split(":")[1]),
        ];
      }

      const now = new Date();
      const eventTime = new Date(now);
      eventTime.setHours(targetHour, targetMinute, 0, 0);

      if (
        eventTime > now &&
        getTimeRemaining(targetHour, targetMinute) > 86340
      ) {
        eventTime.setDate(eventTime.getDate() - 1);
        console.log(
          `Adjusted eventTime for ${n.type} at ${n.time} to previous day: ${eventTime}`
        );
      }

      const secondsSinceEvent = Math.floor(
        (now.getTime() - eventTime.getTime()) / 1000
      );

      console.log(`Filtering ${n.type} at ${n.time}:`);
      console.log(`- Remaining: ${n.remaining}`);
      console.log(`- Seconds since event: ${secondsSinceEvent}`);
      console.log(`- Window seconds: ${windowSeconds}`);
      console.log(
        `- Keep notification: ${(n.remaining > 0 && n.remaining < 86340) || (n.remaining === 0 && secondsSinceEvent < windowSeconds)}`
      );

      return (
        (n.remaining > 0 && n.remaining < 86340) ||
        (n.remaining === 0 && secondsSinceEvent < windowSeconds)
      );
    });

    console.log("Final notifications after filtering:", notifications);
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
        in:fly={{ x: 300, duration: 600, delay: 100 }}
        out:fly={{ x: 300, duration: 600 }}
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
    width: 380px;
    z-index: 1000;
  }

  .toast {
    background: rgba(255, 255, 255, 0.95);
    color: #000000;
    padding: 16px 20px;
    margin: 10px 0;
    border-radius: 10px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    border-left: 6px solid;
    font-size: 16px;
    position: relative;
    overflow: hidden;
    transition: all 0.3s ease;
    backdrop-filter: blur(8px);
  }

  .toast.urgent {
    background-color: rgba(255, 248, 225, 0.95);
    box-shadow: 0 6px 16px rgba(0, 0, 0, 0.4);
    transform: scale(1.05);
  }

  .toast.very-urgent {
    background-color: rgba(255, 235, 238, 0.95);
    box-shadow: 0 8px 20px rgba(255, 0, 0, 0.5);
    transform: scale(1.1);
    animation: pulse 1s infinite;
  }

  @keyframes pulse {
    0% {
      transform: scale(1.05);
    }
    50% {
      transform: scale(1.1);
    }
    100% {
      transform: scale(1.05);
    }
  }

  .notification-content {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .notification-text {
    display: flex;
    align-items: center;
    gap: 10px;
    flex: 1;
  }

  .icon {
    font-size: 20px;
    margin-right: 6px;
  }

  .highlight {
    font-weight: 700;
  }

  .countdown {
    margin-left: 10px;
    font-size: 14px;
    white-space: nowrap;
    font-weight: 600;
  }

  .countdown.urgent {
    font-weight: bold;
    font-size: 16px;
    color: #ff7f50 !important;
  }

  .countdown.very-urgent {
    font-weight: bold;
    font-size: 18px;
    color: #ff0000 !important;
    animation: blink 0.8s infinite;
  }

  @keyframes blink {
    0% {
      opacity: 1;
    }
    50% {
      opacity: 0.4;
    }
    100% {
      opacity: 1;
    }
  }
</style>