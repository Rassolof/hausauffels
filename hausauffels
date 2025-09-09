document.addEventListener("DOMContentLoaded", function () {
  /* ===================== utils ===================== */
  const LOCALE = "de-DE";
  const timeFmt = new Intl.DateTimeFormat(LOCALE, {
    hour: "2-digit",
    minute: "2-digit",
    hour12: false,
  });

  const isValidDate = (d) => d && /^\d{4}-\d{2}-\d{2}$/.test(d);

  // Нормализуем время в 24ч формат "HH:MM".
  // Поддерживает: "9", "9:05", "09:05", "9 AM", "09:05 AM", "12pm", "12:30PM", "21:00"
  function to24h(t) {
    if (!t) return null;
    const s = String(t).trim().toLowerCase();
    const m = s.match(/^(\d{1,2})(?::(\d{2}))?(?::(\d{2}))?\s*(am|pm)?$/i);
    if (!m) return null;

    let hh = parseInt(m[1], 10);
    let mm = m[2] ?? "00";
    const mer = m[4] ? m[4].toLowerCase() : null;

    // Нормализуем минуты
    mm = String(Math.min(59, Math.max(0, parseInt(mm, 10)))).padStart(2, "0");

    // Преобразование AM/PM → 24ч
    if (mer === "am") {
      if (hh === 12) hh = 0; // 12 AM -> 00
    } else if (mer === "pm") {
      if (hh !== 12) hh += 12; // 1..11 PM -> +12
    }
    // Клипуем и паддим
    hh = Math.min(23, Math.max(0, hh));
    const HH = String(hh).padStart(2, "0");

    return `${HH}:${mm}`;
  }

  // Валидность = можем распарсить в 24ч формат
  const isValidTime = (t) => !!to24h(t);

  const parseYMD = (s) => new Date(s + "T00:00:00");
  const ymdLocal = (d) => {
    const y = d.getFullYear();
    const m = String(d.getMonth() + 1).padStart(2, "0");
    const dd = String(d.getDate()).padStart(2, "0");
    return `${y}-${m}-${dd}`;
  };

  const escapeHtml = (x = "") =>
    x.replace(
      /[&<>"']/g,
      (m) =>
        ({
          "&": "&amp;",
          "<": "&lt;",
          ">": "&gt;",
          '"': "&quot;",
          "'": "&#39;",
        }[m])
    );

  // формат строки: "D Monat YYYY | hh:mm bis hh:mm Uhr" — БЕЗ span, с узкими неразрывными пробелами
  function formatDateLineFromISO(startISO, endISO) {
    if (!startISO) return "";
    const start = new Date(
      startISO.includes("T") ? startISO : `${startISO}T00:00:00`
    );

    let dateStr = start
      .toLocaleDateString(LOCALE, {
        day: "numeric",
        month: "long",
        year: "numeric",
      })
      .replace(/(\d+)\.\s/, "$1 ");

    const hasStartTime = /\dT\d{2}:\d{2}/.test(startISO);
    const hasEndTime = !!endISO && /\dT\d{2}:\d{2}/.test(endISO);

    const NNBSP = "\u202F"; // narrow no-break space
    const SEP = `${NNBSP}|${NNBSP}`;
    const BIS = `${NNBSP}bis${NNBSP}`;
    const UHR = `${NNBSP}Uhr`;

    if (hasStartTime || hasEndTime) {
      const s = timeFmt.format(start);
      const e = hasEndTime ? timeFmt.format(new Date(endISO)) : "";
      return e
        ? `${dateStr}${SEP}${s}${BIS}${e}${UHR}`
        : `${dateStr}${SEP}${s}${UHR}`;
    }
    return `${dateStr}`;
  }

  /* ===================== dialog helpers ===================== */
  const dlg = document.getElementById("dialog");
  if (dlg) {
    const dlgCloseBtn = dlg.querySelector(".dialog-close");
    if (dlgCloseBtn) dlgCloseBtn.addEventListener("click", () => dlg.close());
    dlg.addEventListener("click", ({ target, currentTarget }) => {
      if (target === currentTarget) dlg.close();
    });
  }

  function buildDialogCardHTML(title, footerLine, url) {
    const safeTitle = escapeHtml(title || "");
    const urlAttr = url ? ` data-url="${escapeHtml(url)}"` : "";
    // footerLine НЕ экранируем — нам нужны реальные пробелы/разделители
    return `
      <div class="dlg-card">
        <div class="dlg-card__title">${safeTitle}</div>
        <button class="dlg-card__btn"${urlAttr}>Schließen</button>
        <div class="dlg-card__footer">${footerLine ?? ""}</div>
      </div>
    `;
  }

  /* ===================== FULLCALENDAR часть (как было) ===================== */
  const calendarEl = document.getElementById("calendar");
  if (calendarEl) {
    const myEvents = [];
    const dateMap = new Map(); // 'YYYY-MM-DD' -> [events]

    function addRangeToDateMap(startISO, endISO, evt) {
      const startDay = parseYMD(startISO.split("T")[0]);
      const endDay = parseYMD((endISO || startISO).split("T")[0]); // включительно
      for (
        let d = new Date(startDay);
        d <= endDay;
        d.setDate(d.getDate() + 1)
      ) {
        const key = ymdLocal(d);
        if (!dateMap.has(key)) dateMap.set(key, []);
        dateMap.get(key).push(evt);
      }
    }

    document.querySelectorAll(".ec-col-item").forEach((item) => {
      const title =
        item.querySelector(".title")?.textContent?.trim() || "Без названия";
      const startDate = item.querySelector(".start-date")?.textContent?.trim();
      const endDate = item.querySelector(".end-date")?.textContent?.trim();
      const url =
        item.querySelector(".webflow-link")?.getAttribute("href") || "";
      const allDay =
        item.querySelector(".allday")?.textContent?.trim() === "true";
      const className = item.querySelector(".classname")?.textContent?.trim();
      const startTimeRaw = item
        .querySelector(".start-time")
        ?.textContent?.trim();
      const endTimeRaw = item.querySelector(".end-time")?.textContent?.trim();

      if (!isValidDate(startDate)) {
        console.warn("Некорректная дата:", title, startDate);
        return;
      }

      // Нормализуем время (поддержка AM/PM и 24ч)
      const startTime = to24h(startTimeRaw);
      const endTime = to24h(endTimeRaw);

      let startISO = startDate;
      let endISO = isValidDate(endDate) ? endDate : undefined;

      if (!allDay && startTime) {
        startISO = `${startDate}T${startTime}`;
        if (endISO && endTime) endISO = `${endDate || startDate}T${endTime}`;
      }

      const evt = {
        title,
        start: startISO,
        end: endISO,
        url,
        allDay,
        classNames: className ? [className] : [],
        extendedProps: { startTime: startTimeRaw, endTime: endTimeRaw },
      };

      myEvents.push(evt);
      addRangeToDateMap(startISO, endISO, evt);
    });

    const PREV_SVG = `
      <svg width="20" height="20" viewBox="0 0 40 40" fill="none" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
        <path d="M23.002 10L15.5945 19.6296L23.002 29.2593" stroke="currentColor" stroke-width="3" stroke-linecap="round"/>
      </svg>`;
    const NEXT_SVG = `
      <svg width="20" height="20" viewBox="0 0 40 40" fill="none" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
        <path d="M16.998 10L24.4055 19.6296L16.998 29.2593" stroke="currentColor" stroke-width="3" stroke-linecap="round"/>
      </svg>`;

    function applyNavIcons() {
      const prevBtn = calendarEl.querySelector(".fc-prevcustom-button");
      if (prevBtn && prevBtn.innerHTML !== PREV_SVG) {
        prevBtn.innerHTML = PREV_SVG;
        prevBtn.setAttribute("aria-label", "Vorheriger Monat");
      }
      const nextBtn = calendarEl.querySelector(".fc-nextcustom-button");
      if (nextBtn && nextBtn.innerHTML !== NEXT_SVG) {
        nextBtn.innerHTML = NEXT_SVG;
        nextBtn.setAttribute("aria-label", "Nächster Monat");
      }
    }
    function updatePrevDisabled(calendar) {
      const ym = (d) => d.getFullYear() * 12 + d.getMonth();
      const prevBtn = calendarEl.querySelector(".fc-prevcustom-button");
      if (prevBtn) prevBtn.disabled = ym(calendar.getDate()) <= ym(new Date());
    }

    const calendar = new FullCalendar.Calendar(calendarEl, {
      initialView: "dayGridMonth",
      locale: "de",
      timeZone: "Europe/Berlin",
      height: "auto",
      expandRows: false,
      showNonCurrentDates: false,
      fixedWeekCount: false,
      customButtons: {
        prevcustom: {
          text: "",
          hint: "Vorheriger Monat",
          click: () => calendar.prev(),
        },
        nextcustom: {
          text: "",
          hint: "Nächster Monat",
          click: () => calendar.next(),
        },
      },
      headerToolbar: {
        left: "",
        center: "",
        right: "today prevcustom title nextcustom",
      },
      dayHeaderContent: (a) =>
        a.date.toLocaleDateString("de-DE", { weekday: "short" }).slice(0, 2) +
        ".",
      titleFormat: { month: "short" },
      events: myEvents,
      eventDisplay: "none",
      dayCellDidMount(arg) {
        const key = ymdLocal(arg.date);
        if (dateMap.has(key)) arg.el.classList.add("has-event-day");
      },
      dateClick(info) {
        if (!dlg) return;
        const key = info.dateStr;
        if (!dateMap.has(key)) return;

        const sorted = [...dateMap.get(key)].sort(
          (a, b) =>
            new Date(a.start || key + "T00:00:00") -
            new Date(b.start || key + "T00:00:00")
        );

        const html = sorted
          .map((e) => {
            const footer = e.start ? formatDateLineFromISO(e.start, e.end) : "";
            return buildDialogCardHTML(e.title, footer, e.url);
          })
          .join("");

        dlg.innerHTML = `<div class="dialog-content">${html}</div>`;
        dlg.querySelectorAll(".dlg-card__btn").forEach((btn) => {
          const url = btn.getAttribute("data-url");
          btn.addEventListener("click", () =>
            url ? window.open(url, "_blank") : dlg.close()
          );
        });
        dlg.showModal();
      },
      datesSet() {
        applyNavIcons();
        updatePrevDisabled(calendar);
      },
    });

    calendar.render();
    applyNavIcons();
    updatePrevDisabled(calendar);
  }

  /* ===================== SPLIDE: клики по слайдам ===================== */
  document.addEventListener("click", (e) => {
    if (!dlg) return;

    const slide = e.target.closest(".splide__slide");
    if (!slide) return;

    const titleText =
      slide.querySelector(".text-40")?.textContent?.trim() || "";
    const sd = slide.querySelector(".start-date")?.textContent?.trim() || "";
    const ed = slide.querySelector(".end-date")?.textContent?.trim() || "";
    const stRaw = slide.querySelector(".start-time")?.textContent?.trim() || "";
    const etRaw = slide.querySelector(".end-time")?.textContent?.trim() || "";
    const url =
      slide.querySelector(".webflow-link")?.getAttribute("href") || "";

    let startISO = isValidDate(sd) ? sd : null;
    let endISO = isValidDate(ed) ? ed : null;

    const st = to24h(stRaw);
    const et = to24h(etRaw);

    if (startISO && st) startISO = `${sd}T${st}`;
    if (endISO && et) endISO = `${ed || sd}T${et}`;

    const footerLine = startISO ? formatDateLineFromISO(startISO, endISO) : "";

    const html = buildDialogCardHTML(titleText, footerLine, url);
    dlg.innerHTML = `<div class="dialog-content">${html}</div>`;
    dlg.querySelectorAll(".dlg-card__btn").forEach((btn) => {
      const url = btn.getAttribute("data-url");
      btn.addEventListener("click", () =>
        url ? window.open(url, "_blank") : dlg.close()
      );
    });
    dlg.showModal();
  });
});
// hidden before date //
// document.addEventListener("DOMContentLoaded", () => {
//   const list = document.querySelector(".splide__list");
//   if (!list) return;

//   const todayYMD = new Date().toISOString().split("T")[0]; // YYYY-MM-DD

//   Array.from(list.children).forEach((slide) => {
//     const sd = slide.querySelector(".start-date")?.textContent?.trim();
//     if (sd && sd < todayYMD) slide.remove(); // <-- удаляем, а не скрываем
//   });
// });
// Button animation
document.querySelectorAll(".wrapper-button").forEach((block) => {
  const trigger = block.querySelector(".button, .navigation_link");
  const close = block.querySelector(".is-close");

  if (trigger) {
    trigger.addEventListener("click", () => {
      trigger.classList.add("is-active");
    });
  }

  if (close && trigger) {
    close.addEventListener("click", (e) => {
      e.stopPropagation();
      trigger.classList.remove("is-active");
    });
  }
});

// Copy text
document.querySelectorAll(".copy_text").forEach((el) => {
  el.addEventListener("click", () => {
    const text = el.innerText; // сохранит переносы строк
    navigator.clipboard.writeText(text);
  });
});
