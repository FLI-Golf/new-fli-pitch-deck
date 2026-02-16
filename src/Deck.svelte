<script>
  export let infoDesk;
  export let infoMobile;
  import MediaQuery from "./MediaQuery.svelte";
  import { onDestroy, tick } from "svelte";

  let index = 0;
  let speaking = false;
  let exporting = false;
  let synth = typeof window !== 'undefined' ? window.speechSynthesis : null;

  function nextSlide(items) {
      if (index < items.length - 1) {
          index += 1;
      }
      stopSpeech();
  }

  function prevSlide() {
      if (index > 0) {
          index -= 1;
      }
      stopSpeech();
  }

  function getSlideText(items) {
      let el = document.createElement('div');
      el.innerHTML = items[index].text;
      let text = items[index].title ? items[index].title + '. ' : '';
      text += el.textContent || el.innerText || '';
      return text;
  }

  function toggleSpeech(items) {
      if (!synth) return;
      if (speaking) {
          stopSpeech();
      } else {
          let text = getSlideText(items);
          let utterance = new SpeechSynthesisUtterance(text);
          utterance.rate = 0.9;
          utterance.onend = () => { speaking = false; };
          synth.speak(utterance);
          speaking = true;
      }
  }

  function stopSpeech() {
      if (synth) {
          synth.cancel();
          speaking = false;
      }
  }

  async function exportPDF(items) {
      stopSpeech();
      exporting = true;
      await tick();

      const savedIndex = index;
      // 16:9 widescreen: 13.333 x 7.5 inches
      const pdfW = 13.333;
      const pdfH = 7.5;
      const pdf = new window.jspdf.jsPDF({ orientation: 'landscape', unit: 'in', format: [pdfW, pdfH] });

      // Create an offscreen container at fixed pixel size for consistent rendering
      const renderW = 1280;
      const renderH = 720;
      const offscreen = document.createElement('div');
      offscreen.style.cssText = 'position:fixed;left:-9999px;top:0;width:' + renderW + 'px;height:' + renderH + 'px;overflow:hidden;z-index:-1;';
      document.body.appendChild(offscreen);

      for (let i = 0; i < items.length; i++) {
          // Build slide HTML in the offscreen container
          const slideEl = document.createElement('div');
          slideEl.style.cssText = 'width:' + renderW + 'px;height:' + renderH + 'px;background:#294582;color:white;text-align:center;font-size:16px;font-family:-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,sans-serif;display:flex;flex-direction:column;box-sizing:border-box;border:2px solid rgba(255,255,255,0.2);border-radius:8px;overflow:hidden;';

          // Content area
          const contentEl = document.createElement('div');
          contentEl.style.cssText = 'flex:1;display:flex;flex-direction:column;align-items:center;justify-content:center;padding:30px 50px 20px 50px;overflow:hidden;';

          if (items[i].title) {
              const h1 = document.createElement('h1');
              h1.textContent = items[i].title;
              h1.style.cssText = 'text-transform:uppercase;font-size:28px;font-weight:100;font-family:Nanum Myeongjo,serif;margin:5px 0 10px 0;width:100%;';
              contentEl.appendChild(h1);
          }

          const bodyDiv = document.createElement('div');
          bodyDiv.innerHTML = items[i].text;
          bodyDiv.style.cssText = 'width:100%;';
          // Fix images to use absolute pixel sizes for PDF
          const imgs = bodyDiv.querySelectorAll('img');
          imgs.forEach(img => {
              if (img.style.width && img.style.width.includes('vmin')) {
                  img.style.width = '100px';
              }
          });
          contentEl.appendChild(bodyDiv);
          slideEl.appendChild(contentEl);

          // Footer
          const footerEl = document.createElement('div');
          footerEl.style.cssText = 'display:flex;align-items:center;justify-content:center;gap:10px;padding:8px 0;border-top:1px solid rgba(255,255,255,0.15);background:rgba(0,0,0,0.15);';
          footerEl.innerHTML = '<img src="FGL_logo.png" style="height:20px;width:auto;" alt="FGL"/><span style="color:rgba(255,255,255,0.6);font-size:11px;">Slide ' + (i + 1) + ' of ' + items.length + '</span>';
          slideEl.appendChild(footerEl);

          offscreen.innerHTML = '';
          offscreen.appendChild(slideEl);

          // Wait for images to load
          const slideImgs = slideEl.querySelectorAll('img');
          await Promise.all(Array.from(slideImgs).map(img => {
              if (img.complete) return Promise.resolve();
              return new Promise(resolve => { img.onload = resolve; img.onerror = resolve; });
          }));

          const canvas = await window.html2canvas(slideEl, {
              width: renderW,
              height: renderH,
              scale: 2,
              backgroundColor: '#294582',
              useCORS: true
          });

          const imgData = canvas.toDataURL('image/jpeg', 0.95);
          if (i > 0) pdf.addPage([pdfW, pdfH], 'landscape');
          pdf.addImage(imgData, 'JPEG', 0, 0, pdfW, pdfH);
      }

      document.body.removeChild(offscreen);
      pdf.save('FLI_Golf_League_Deck.pdf');

      index = savedIndex;
      exporting = false;
      await tick();
  }

  function handleKeydown(e) {
      if (exporting) return;
      if (e.key === 'ArrowRight' || e.key === ' ') {
          e.preventDefault();
          nextSlide(infoDesk);
      } else if (e.key === 'ArrowLeft') {
          e.preventDefault();
          prevSlide();
      }
  }

  onDestroy(() => {
      stopSpeech();
  });
</script>

<svelte:window on:keydown={handleKeydown} />

<style>
  .slide-wrapper {
      background-color: #1a2d5a;
      width: 100vw;
      height: 100vh;
      display: flex;
      flex-direction: column;
      box-sizing: border-box;
      padding: 16px;
  }

  .slide {
      background-color: #294582;
      color: white;
      text-align: center;
      font-size: 2.2vmin;
      font-weight: 10;
      flex: 1;
      border: 2px solid rgba(255, 255, 255, 0.2);
      border-radius: 8px;
      box-sizing: border-box;
      display: flex;
      flex-direction: column;
      position: relative;
      overflow: hidden;
  }

  .slide-content {
      flex: 1;
      overflow: hidden;
      padding: 2vh 4vw 1vh 4vw;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
  }

  .slide-mobile {
      font-size: 1em;
  }

  .slide-mobile .slide-content {
      padding: 2vh 3vw 1vh 3vw;
  }

  .deskh1 {
      text-transform: uppercase;
      font-size: 1.5em;
      font-weight: 100;
      font-family: 'Nanum Myeongjo', serif;
      margin: 0.5vh 0 0.5vh 0;
      width: 100%;
  }

  .mobileh1 {
      text-transform: uppercase;
      font-family: 'Nanum Myeongjo', serif;
      margin: 1vh 0 1vh 0;
      width: 100%;
  }

  .deskpad {
      padding-left: 3vw;
      padding-right: 3vw;
      width: 100%;
  }

  .mobilepad {
      padding-left: 2vw;
      padding-right: 2vw;
      width: 100%;
  }

  .footer {
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 10px;
      padding: 8px 0;
      border-top: 1px solid rgba(255, 255, 255, 0.15);
      background-color: rgba(0, 0, 0, 0.15);
  }

  .footer img {
      height: 24px;
      width: auto;
  }

  .footer span {
      color: rgba(255, 255, 255, 0.6);
      font-size: 12px;
      letter-spacing: 0.05em;
  }

  .controls {
      position: fixed;
      bottom: 20px;
      right: 20px;
      display: flex;
      gap: 8px;
      align-items: center;
      z-index: 100;
  }

  .controls button {
      background: rgba(0, 0, 0, 0.5);
      color: white;
      border: 1px solid rgba(255, 255, 255, 0.3);
      border-radius: 6px;
      padding: 8px 14px;
      font-size: 14px;
      cursor: pointer;
      display: inline-flex !important;
      align-items: center;
      gap: 5px;
      transition: background 0.2s;
      margin: 0;
      white-space: nowrap;
  }

  .controls button:hover {
      background: rgba(0, 0, 0, 0.75);
  }

  .controls button:disabled {
      opacity: 0.3;
      cursor: default;
  }

  .controls button:disabled:hover {
      background: rgba(0, 0, 0, 0.5);
  }

  .speaking {
      background: rgba(200, 60, 60, 0.7) !important;
  }


</style>

<MediaQuery query="(max-width: 800px)" let:matches>
  {#if matches}
    <div class="slide-wrapper">
      <div class="slide slide-mobile">
        <div class="slide-content">
          <h1 class="mobileh1">{infoMobile[index].title}</h1>
          <p class="mobilepad">{@html infoMobile[index].text}</p>
        </div>
        <div class="footer">
          <img src="FGL_logo.png" alt="FGL" />
          <span>Slide {index + 1} of {infoMobile.length}</span>
        </div>
      </div>
    </div>
    <div class="controls">
      <button on:click|stopPropagation={prevSlide} disabled={index === 0}><svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m15 18-6-6 6-6"/></svg> Prev</button>
      <button on:click|stopPropagation={() => nextSlide(infoMobile)} disabled={index === infoMobile.length - 1}>Next <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m9 18 6-6-6-6"/></svg></button>
      <button on:click|stopPropagation={() => toggleSpeech(infoMobile)} class:speaking>{#if speaking}<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M11 4.702a.705.705 0 0 0-1.203-.498L6.413 7.587A1.4 1.4 0 0 1 5.416 8H3a1 1 0 0 0-1 1v6a1 1 0 0 0 1 1h2.416a1.4 1.4 0 0 1 .997.413l3.383 3.384A.705.705 0 0 0 11 19.298z"/><line x1="22" y1="9" x2="16" y2="15"/><line x1="16" y1="9" x2="22" y2="15"/></svg> Stop{:else}<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M11 4.702a.705.705 0 0 0-1.203-.498L6.413 7.587A1.4 1.4 0 0 1 5.416 8H3a1 1 0 0 0-1 1v6a1 1 0 0 0 1 1h2.416a1.4 1.4 0 0 1 .997.413l3.383 3.384A.705.705 0 0 0 11 19.298z"/><path d="M16 9a5 5 0 0 1 0 6"/><path d="M19.364 18.364a9 9 0 0 0 0-12.728"/></svg> Read{/if}</button>
      <button on:click|stopPropagation={() => exportPDF(infoMobile)} disabled={exporting}><svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M15 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7Z"/><path d="M14 2v4a2 2 0 0 0 2 2h4"/><path d="M12 18v-6"/><path d="m9 15 3 3 3-3"/></svg> {exporting ? 'Exporting...' : 'PDF'}</button>
    </div>
  {:else}
    <div class="slide-wrapper">
      <div class="slide">
        <div class="slide-content">
          <h1 class="deskh1">{infoDesk[index].title}</h1>
          <p class="deskpad">{@html infoDesk[index].text}</p>
        </div>
        <div class="footer">
          <img src="FGL_logo.png" alt="FGL" />
          <span>Slide {index + 1} of {infoDesk.length}</span>
        </div>
      </div>
    </div>
    <div class="controls">
      <button on:click|stopPropagation={prevSlide} disabled={index === 0}><svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m15 18-6-6 6-6"/></svg> Prev</button>
      <button on:click|stopPropagation={() => nextSlide(infoDesk)} disabled={index === infoDesk.length - 1}>Next <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m9 18 6-6-6-6"/></svg></button>
      <button on:click|stopPropagation={() => toggleSpeech(infoDesk)} class:speaking>{#if speaking}<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M11 4.702a.705.705 0 0 0-1.203-.498L6.413 7.587A1.4 1.4 0 0 1 5.416 8H3a1 1 0 0 0-1 1v6a1 1 0 0 0 1 1h2.416a1.4 1.4 0 0 1 .997.413l3.383 3.384A.705.705 0 0 0 11 19.298z"/><line x1="22" y1="9" x2="16" y2="15"/><line x1="16" y1="9" x2="22" y2="15"/></svg> Stop{:else}<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M11 4.702a.705.705 0 0 0-1.203-.498L6.413 7.587A1.4 1.4 0 0 1 5.416 8H3a1 1 0 0 0-1 1v6a1 1 0 0 0 1 1h2.416a1.4 1.4 0 0 1 .997.413l3.383 3.384A.705.705 0 0 0 11 19.298z"/><path d="M16 9a5 5 0 0 1 0 6"/><path d="M19.364 18.364a9 9 0 0 0 0-12.728"/></svg> Read{/if}</button>
      <button on:click|stopPropagation={() => exportPDF(infoDesk)} disabled={exporting}><svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M15 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7Z"/><path d="M14 2v4a2 2 0 0 0 2 2h4"/><path d="M12 18v-6"/><path d="m9 15 3 3 3-3"/></svg> {exporting ? 'Exporting...' : 'PDF'}</button>
    </div>
  {/if}
</MediaQuery>
