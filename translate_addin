// ==UserScript==
// @name         Gemini AI Inline Translator (Popup)
// @namespace    http://tampermonkey.net/
// @version      2.8
// @description  Dịch văn bản bôi đen bằng Google Gemini API, có phím tắt, sửa lỗi ký tự, đảm bảo chỉ dịch một lần và có nút đóng. Cải thiện chất lượng dịch. Hỗ trợ phân tích từ vựng (Alt + M) hiển thị trong popup và dịch nhanh (Alt + T).
// @author       Your Name
// @match        *://*/*
// @grant        GM_xmlhttpRequest
// @grant        GM_addStyle
// ==/UserScript==
(function () {
    'use strict';

    // Cấu hình API
    const API_CONFIG = {
        providers: {
            gemini: {
                url: (apiKey) => `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-lite-preview-02-05:generateContent?key=${apiKey}`,
                headers: {
                    'Content-Type': 'application/json'
                },
                body: (prompt) => ({
                    contents: [{
                        role: "user",
                        parts: [{ text: prompt }]
                    }],
                    generationConfig: { temperature: 0.7 }
                }),
                responseParser: (response) => {
                    if (!response.candidates || response.candidates.length === 0) {
                        throw new Error('Gemini API: No candidates in response');
                    }
                    if (!response.candidates[0].content?.parts?.[0]?.text) {
                        throw new Error('Gemini API: Invalid response format');
                    }
                    return response.candidates[0].content.parts[0].text;
                }
            },
            openai: {
                url: () => 'https://api.groq.com/openai/v1/chat/completions',
                headers: (apiKey) => ({
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${apiKey}`
                }),
                body: (prompt) => ({
                    model: "llama-3.3-70b-versatile",
                    messages: [{ role: "user", content: prompt }],
                    temperature: 0.7
                }),
                responseParser: (response) => response.choices?.[0]?.message?.content
            }
        },
        currentProvider: 'gemini',
        //apiKey: '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', // OpenAI completion api
        apiKey: 'AIzaSyBUB7X9HBanxuXQgxRX6q71CMSvm_LMnDU', // Gemini
        maxRetries: 3,
        retryDelay: 1000,
        rateLimit: {
            maxRequests: 5,
            perMilliseconds: 10000
        }
    };

    // Biến trạng thái
    let isTranslating = false;
    let requestQueue = [];
    let requestCount = 0;
    let lastRequestTime = 0;

     // Add CSS for the draggable popup
    GM_addStyle(`
        .draggable {
          cursor: move;
        }
    `);

    // Hàm gọi API Gemini để dịch văn bản
     async function translateText(text, targetElement, isAdvanced = false, displaySimple = false) {
        // Thêm vào hàng đợi và xử lý tuần tự
        return new Promise((resolve) => {
            requestQueue.push(async () => {
                try {
                    if (isTranslating) return;
                    isTranslating = true;

                    // Kiểm tra rate limiting
                    const now = Date.now();
                    if (now - lastRequestTime < API_CONFIG.rateLimit.perMilliseconds) {
                        if (requestCount >= API_CONFIG.rateLimit.maxRequests) {
                            const delay = API_CONFIG.rateLimit.perMilliseconds - (now - lastRequestTime);
                            await new Promise(res => setTimeout(res, delay));
                            requestCount = 0;
                        }
                    } else {
                        requestCount = 0;
                        lastRequestTime = now;
                    }

                    // Tạo prompt
                    //let prompt = `Translate to Vietnamese naturally: "${text}"`;
                    let prompt = `Translate "${text}" to Vietnamese with:
                                  - Strict adherence to original context and nuances
                                  - Natural fluency as spoken by native speakers
                                  - No added explanations/interpretations
                                  - 1:1 structure preservation for terms/proper nouns
                                  Output only the translation without quotation marks`;
                    if (isAdvanced) {
                        prompt = `Translate to Vietnamese and analyze keywords then give example for each keyword: "${text}"`;
                    }

                    const provider = API_CONFIG.providers[API_CONFIG.currentProvider];
                    let translatedText = '';
                    let attempts = 0;

                    // Retry logic với exponential backoff
                    while (attempts < API_CONFIG.maxRetries) {
                        try {
                            translatedText = await new Promise((resolve, reject) => {
                                GM_xmlhttpRequest({
                                    method: 'POST',
                                    url: provider.url(API_CONFIG.apiKey),
                                    headers: typeof provider.headers === 'function' ? provider.headers(API_CONFIG.apiKey) : provider.headers,
                                    data: JSON.stringify(provider.body(prompt)),
                                    onload: function(response) {
                                        if (response.status >= 200 && response.status < 300) {
                                            const result = JSON.parse(response.responseText);
                                            const text = provider.responseParser(result);
                                            text ? resolve(text) : reject('Invalid response format');
                                        } else if (response.status === 429) {
                                            reject('Rate limit exceeded');
                                        } else {
                                            reject(`API Error: ${response.status}`);
                                        }
                                    },
                                    onerror: function(error) {
                                        reject(`Connection error: ${error}`);
                                    }
                                });
                            });

                            requestCount++;
                            break; // Thoát vòng lặp nếu thành công
                        } catch (error) {
                            attempts++;
                            if (attempts >= API_CONFIG.maxRetries) throw error;
                            await new Promise(res =>
                                setTimeout(res, API_CONFIG.retryDelay * Math.pow(2, attempts))
                            );
                        }
                    }

                    // Hiển thị kết quả
                    if (isAdvanced) {
                        displayPopup(translatedText, text);
                    } else if (displaySimple) {
                        displaySimplePopup(text, translatedText);
                    } else {
                        showTranslationBelow(targetElement, translatedText);
                    }
                } catch (error) {
                    console.error('Translation failed:', error);
                    const errorMessage = error instanceof Error ? error.message : String(error);
                    showErrorBelow(targetElement,
                        errorMessage.includes('Rate limit') ? 'Vui lòng chờ giữa các lần dịch' :
                        errorMessage.includes('Gemini API') ? 'Lỗi Gemini: ' + errorMessage :
                        errorMessage.includes('API Key') ? 'Lỗi xác thực API' :
                        'Lỗi dịch thuật: ' + errorMessage);
                } finally {
                    isTranslating = false;
                    requestQueue.shift();
                    if (requestQueue.length > 0) requestQueue[0]();
                }
            });

            if (!isTranslating && requestQueue.length === 1) {
                requestQueue[0]();
            }
        });
    }


      // Hàm hiển thị bản dịch ngay bên dưới đoạn văn bản được bôi đen
   function showTranslationBelow(targetElement, translatedText) {
       // Tìm đoạn văn cuối cùng được bôi đen
        const selection = window.getSelection();
         const lastSelectedNode = selection.focusNode;
        let lastSelectedParagraph = lastSelectedNode.parentElement;

        // Đảm bảo phần tử cuối cùng là một đoạn văn
        while (lastSelectedParagraph && lastSelectedParagraph.tagName !== 'P') {
           lastSelectedParagraph = lastSelectedParagraph.parentElement;
        }

         // Nếu không tìm thấy, sử dụng targetElement (đoạn đầu tiên)
         if (!lastSelectedParagraph) {
            lastSelectedParagraph = targetElement;
        }

        // Kiểm tra xem đã có bản dịch nào được hiển thị chưa
        if (lastSelectedParagraph.nextElementSibling && lastSelectedParagraph.nextElementSibling.classList.contains('translation-div')) {
           return; // Nếu đã có bản dịch, không hiển thị thêm
       }

        const translationDiv = document.createElement('div');
        translationDiv.classList.add('translation-div'); // Thêm class để nhận diện
        translationDiv.style.marginTop = '10px';
         translationDiv.style.padding = '10px';
        translationDiv.style.backgroundColor = '#f0f0f0';
        translationDiv.style.borderLeft = '3px solid #4CAF50';
         translationDiv.style.color = '#333';
       translationDiv.style.position = 'relative'; // Thêm thuộc tính position: relative

       // Thiết lập font chữ và cỡ chữ
        translationDiv.style.fontFamily = 'SF Pro Rounded, sans-serif';
        translationDiv.style.fontSize = '16px'; // Cỡ chữ 16px (có thể điều chỉnh từ 15-17px)

       translationDiv.textContent = `Dịch: ${translatedText}`;


        // Thêm nút đóng (nút "x")
        const closeButton = document.createElement('span');
        closeButton.textContent = 'x';
        closeButton.style.position = 'absolute';
       closeButton.style.top = '5px';
        closeButton.style.right = '5px';
       closeButton.style.cursor = 'pointer';
        closeButton.style.color = '#999';
       closeButton.style.fontSize = '14px';
        closeButton.style.fontWeight = 'bold';

        closeButton.addEventListener('click', function () {
           translationDiv.remove();
        });

        translationDiv.appendChild(closeButton);

       lastSelectedParagraph.parentNode.insertBefore(translationDiv, lastSelectedParagraph.nextSibling);
    }



      // Function to display the summary on the webpage with dynamic width and scrollable content
    function displayPopup(translatedText, originalText) {
         const summaryDiv = document.createElement('div');
        summaryDiv.classList.add('draggable');
         summaryDiv.style.position = 'fixed';
        summaryDiv.style.top = '50%';
        summaryDiv.style.left = '50%';
        summaryDiv.style.transform = 'translate(-50%, -50%)';
         summaryDiv.style.backgroundColor = '#fff';
        summaryDiv.style.border = '1px solid #ccc';
        summaryDiv.style.padding = '20px';
         summaryDiv.style.zIndex = '10000';
         summaryDiv.style.width = '700px';    // Wider width
        summaryDiv.style.maxHeight = '500px';   // Lower max height
        summaryDiv.style.boxShadow = '0 0 10px rgba(0, 0, 0, 0.1)';
        summaryDiv.style.borderRadius = '15px';
        summaryDiv.style.fontFamily = 'SF Pro Rounded, Arial, sans-serif';
        summaryDiv.style.fontSize = '16px'; // Font size 16px
        summaryDiv.style.display = 'flex';
        summaryDiv.style.flexDirection = 'column';
         summaryDiv.style.overflow = 'hidden';

         // Add summary section with scrollable content
        const summarySection = document.createElement('div');
       const cleanedSummary = translatedText.replace(/(\*\*)(.*?)\1/g, '<b>$2</b>'); // Remove ** markers
        const formattedSummary = cleanedSummary.split('<br>').map(line => {
            if (line.startsWith('<b>KEYWORD</b>:')) {
                 return `<h4 style="margin-bottom: 5px;">${line}</h4>`;
             } else if (line.startsWith('+ Định nghĩa:')) {
                const parts = line.split(':');
                if (parts.length > 1) {
                    const definition = parts.slice(1).join(':').trim();
                    const definitionParts = definition.split('<br> - Bản dịch định nghĩa:');
                    if (definitionParts.length === 2) {
                        return `<p style="margin-left: 20px; margin-bottom: 5px; white-space: pre-wrap; word-wrap: break-word; text-align: justify;">+ Định nghĩa:<br>${definitionParts[0].trim()}</p><p style="margin-left: 20px; margin-bottom: 10px; white-space: pre-wrap; word-wrap: break-word; text-align: justify;"> - Bản dịch định nghĩa: ${definitionParts[1].trim()}</p>`;
                   }
                }
                return `<p style="margin-left: 20px; margin-bottom: 10px; white-space: pre-wrap; word-wrap: break-word; text-align: justify;">${line}</p>`;
             }  else if (line.startsWith('+ Ví dụ:')) {
                const parts = line.split(':');
                 if (parts.length > 1) {
                    const example = parts.slice(1).join(':').trim();
                     const exampleParts = example.split('<br> - Bản dịch ví dụ:');
                     if (exampleParts.length === 2) {
                       // Replace example with a sentence from originalText if available
                         const keywordMatch = line.match(/<b>KEYWORD<\/b>:\s*(\w+)/i);
                         let exampleFromText = null;
                      if (keywordMatch)
                      {
                           const keyword = keywordMatch[1];
                            const regex = new RegExp(`\\b${keyword}\\b`, 'i');
                           const sentences = originalText.split(/[.?!]/).filter(sentence => regex.test(sentence));
                           if (sentences.length > 0)
                             exampleFromText = sentences[0].trim() + '.';

                     }
                     const displayExample = exampleFromText ? exampleFromText : exampleParts[0].trim();
                       return `<p style="margin-left: 20px; margin-bottom: 5px; white-space: pre-wrap; word-wrap: break-word; text-align: justify;">+ Ví dụ: ${displayExample}</p><p style="margin-left: 20px; margin-bottom: 10px; white-space: pre-wrap; word-wrap: break-word; text-align: justify;"> - Bản dịch ví dụ: ${exampleParts[1].trim()}</p>`;
                    }
                 }

                 return `<p style="margin-left: 20px; margin-bottom: 10px; white-space: pre-wrap; word-wrap: break-word; text-align: justify;">${line}</p>`;
             } else {
                return `<p style="margin-left: 20px; margin-bottom: 10px; white-space: pre-wrap; word-wrap: break-word; text-align: justify;">${line}</p>`;
           }
        }).join('');

        summarySection.innerHTML = `<h3 style="color: #333;">Dịch</h3><div style="overflow-y: auto; max-height: 400px; color: #555; font-size: 16px;">${formattedSummary}</div>`; // Font size 16px and lower max-height
        summaryDiv.appendChild(summarySection);


         // Add close button
        const closeButton = document.createElement('button');
       closeButton.innerText = 'Đóng';
        closeButton.style.marginTop = '10px';
       closeButton.style.padding = '8px 16px';
        closeButton.style.backgroundColor = '#ff4444';
        closeButton.style.color = '#fff';
        closeButton.style.border = 'none';
       closeButton.style.borderRadius = '4px';
        closeButton.style.cursor = 'pointer';
       closeButton.onclick = () => summaryDiv.remove();
        summaryDiv.appendChild(closeButton);

        makeDraggable(summaryDiv);
        document.body.appendChild(summaryDiv);
   }


    // Function to display the simple translation popup
    function displaySimplePopup(originalText, translatedText) {
        const simplePopupDiv = document.createElement('div');
        simplePopupDiv.classList.add('draggable');
        simplePopupDiv.style.position = 'fixed';
        simplePopupDiv.style.top = '50%';
        simplePopupDiv.style.left = '50%';
        simplePopupDiv.style.transform = 'translate(-50%, -50%)';
        simplePopupDiv.style.backgroundColor = '#fff';
        simplePopupDiv.style.border = '1px solid #ccc';
        simplePopupDiv.style.padding = '20px';
        simplePopupDiv.style.zIndex = '10000';
        simplePopupDiv.style.width = '400px'; // Wider width for simple popup
        simplePopupDiv.style.maxHeight = '400px'; // Lower max height for simple popup
        simplePopupDiv.style.boxShadow = '0 0 10px rgba(0, 0, 0, 0.1)';
        simplePopupDiv.style.borderRadius = '15px';
        simplePopupDiv.style.fontFamily = 'SF Pro Rounded, Arial, sans-serif';
        simplePopupDiv.style.fontSize = '16px'; // Font size 16px
        simplePopupDiv.style.display = 'flex';
        simplePopupDiv.style.flexDirection = 'column';
        simplePopupDiv.style.overflow = 'hidden'; // Keep overflow hidden for container, content will scroll

        // Translated Text Section (Only translation, no original text)
        const translatedTextSection = document.createElement('div');
        translatedTextSection.innerHTML = `<div style="white-space: pre-wrap; word-wrap: break-word; text-align: justify; font-size: 16px; overflow-y: auto; max-height: 350px; padding-right: 10px;">${translatedText}</div>`; // Increased font size, added scroll and padding, lower max-height
        simplePopupDiv.appendChild(translatedTextSection);

        // Close Button
        const closeButton = document.createElement('button');
        closeButton.innerText = 'Đóng';
        closeButton.style.marginTop = '10px';
        closeButton.style.padding = '8px 16px';
        closeButton.style.backgroundColor = '#ff4444';
        closeButton.style.color = '#fff';
        closeButton.style.border = 'none';
        closeButton.style.borderRadius = '4px';
        closeButton.style.cursor = 'pointer';
        closeButton.onclick = () => simplePopupDiv.remove();
        simplePopupDiv.appendChild(closeButton);

        makeDraggable(simplePopupDiv);
        document.body.appendChild(simplePopupDiv);
    }


    // Cache các bản dịch
    const translationCache = new Map();
    const CACHE_EXPIRATION = 300000; // 5 phút

    // Hàm thêm vào cache
    function addToCache(original, translated) {
        translationCache.set(original, {
            text: translated,
            timestamp: Date.now()
        });
    }

    // Hàm kiểm tra cache
    function checkCache(text) {
        const entry = translationCache.get(text);
        if (entry && Date.now() - entry.timestamp < CACHE_EXPIRATION) {
            return entry.text;
        }
        translationCache.delete(text);
        return null;
    }

    // Hàm hiển thị lỗi
  function showErrorBelow(targetElement, errorMessage) {
        // Tìm đoạn văn cuối cùng được bôi đen (tương tự như trên)
        const selection = window.getSelection();
       const lastSelectedNode = selection.focusNode;
        let lastSelectedParagraph = lastSelectedNode.parentElement;

        while (lastSelectedParagraph && lastSelectedParagraph.tagName !== 'P') {
            lastSelectedParagraph = lastSelectedParagraph.parentElement;
         }

        if (!lastSelectedParagraph) {
            lastSelectedParagraph = targetElement;
        }

        if (lastSelectedParagraph.nextElementSibling && lastSelectedParagraph.nextElementSibling.classList.contains('translation-div')) {
            return;
       }

       const errorDiv = document.createElement('div');
        errorDiv.classList.add('translation-div');
        errorDiv.style.marginTop = '10px';
        errorDiv.style.padding = '10px';
       errorDiv.style.backgroundColor = '#fdd';
         errorDiv.style.borderLeft = '3px solid #faa';
        errorDiv.style.color = '#a00';
         errorDiv.style.fontFamily = 'SF Pro Rounded Medium, sans-serif';
        errorDiv.style.fontSize = '16px';
        errorDiv.style.position = 'relative'; // Thêm thuộc tính position: relative

       errorDiv.textContent = errorMessage;

        // Thêm nút đóng (nút "x")
         const closeButton = document.createElement('span');
        closeButton.textContent = 'x';
        closeButton.style.position = 'absolute';
        closeButton.style.top = '5px';
        closeButton.style.right = '5px';
        closeButton.style.cursor = 'pointer';
        closeButton.style.color = '#999';
        closeButton.style.fontSize = '14px';
         closeButton.style.fontWeight = 'bold';

        closeButton.addEventListener('click', function () {
            errorDiv.remove();
       });

       errorDiv.appendChild(closeButton);

       lastSelectedParagraph.parentNode.insertBefore(errorDiv, lastSelectedParagraph.nextSibling);
    }

    // Function to make the summary popup draggable
    function makeDraggable(element) {
         let pos1 = 0, pos2 = 0, pos3 = 0, pos4 = 0;
        element.onmousedown = dragMouseDown;

        function dragMouseDown(e) {
            e = e || window.event;
            e.preventDefault();
             // get the mouse cursor position at startup
           pos3 = e.clientX;
            pos4 = e.clientY;
            document.onmouseup = closeDragElement;
           // call a function whenever the cursor moves
            document.onmousemove = elementDrag;
        }

       function elementDrag(e) {
           e = e || window.event;
            e.preventDefault();
            // calculate the new cursor position:
           pos1 = pos3 - e.clientX;
            pos2 = pos4 - e.clientY;
            pos3 = e.clientX;
            pos4 = e.clientY;
             // set the element's new position:
             element.style.top = (element.offsetTop - pos2) + "px";
            element.style.left = (element.offsetLeft - pos1) + "px";
       }

       function closeDragElement() {
            // stop moving when mouse button is released:
            document.onmouseup = null;
           document.onmousemove = null;
        }
    }


     // Lắng nghe sự kiện phím tắt (Ctrl + Q hoặc Cmd + Q, Alt + Q và Alt + T)
    document.addEventListener('keydown', function (event) {
        const selection = window.getSelection();
         const selectedText = selection.toString().trim();
        if (!selectedText) return;

        const targetElement = selection.anchorNode.parentElement;

        if ((event.ctrlKey || event.metaKey) && event.key === 'q') {
             event.preventDefault(); // Ngăn chặn hành động mặc định của trình duyệt
            translateText(selectedText, targetElement);
        }
        else if (event.altKey && event.key === 'q') {
           event.preventDefault();
            translateText(selectedText, targetElement, true); // Dịch nâng cao khi ấn Alt + Q
        }
        else if (event.altKey && event.key === 't') {
            event.preventDefault();
            translateText(selectedText, targetElement, false, true); // Dịch nhanh popup khi ấn Alt + T
        }
    });


})();

