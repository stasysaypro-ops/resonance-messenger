/* SNS CHAT v2.42 | lazy media, cached search, shared reactions | RESONANCE / RusFF | Forum ID: 4 | UTF-8 */
(function(window, document) {
  "use strict";
  if (!/\/viewtopic\.php$/i.test(window.location.pathname) || window.__SNS_V241_BOOT__) return;
  window.__SNS_V241_BOOT__ = true;
  var bootAttempts = 0;
  function boot() {
    if (!window.jQuery) {
      if (++bootAttempts < 20) setTimeout(boot, 250); else window.__SNS_V241_BOOT__ = false;
      return;
    }
    var $ = window.jQuery;
    var forum = Number(window.ForumID) || 0;
    if (!forum) {
      var links = document.querySelectorAll('#pun-crumbs1 a[href*="viewforum.php?id="],#pun-crumbs2 a[href*="viewforum.php?id="]');
      var match = links.length && (links[links.length - 1].getAttribute("href") || "").match(/[?&]id=(\d+)/);
      forum = match ? Number(match[1]) : 0;
    }
    if (forum !== 4 || !document.querySelector("#pun-viewtopic .topic")) return;
    var url = new URL(window.location.href);
    if (Number(url.searchParams.get("p")) > 1) {
      url.searchParams.delete("p");
      url.hash = "";
      window.location.replace(url.toString());
      return;
    }
    if (window.__SNS_V241_STARTED__) return;
    window.__SNS_V241_STARTED__ = true;
    run($);
  }
  if (document.readyState === "loading") document.addEventListener("DOMContentLoaded", boot, {
    once: true
  }); else boot();
  function run(jQuery) {
    var $ = jQuery;
    var snsQueue = [];
    var snsActive = 0;
    var snsNextRequestAt = 0;
    var snsBlockedUntil = 0;
    var snsPumpTimer = null;
    var snsStats = {
      build: "v2.42",
      requests: 0,
      rateLimits: 0
    };
    window.SNSPerformanceState = function() {
      return {
        build: snsStats.build,
        requests: snsStats.requests,
        queued: snsQueue.length,
        active: snsActive,
        cooldownMs: Math.max(0, snsBlockedUntil - Date.now()),
        hidden: document.hidden,
        posts: document.querySelectorAll("#sns-chat-shell>.topic>.post").length
      };
    };
    function SNSRequest(options, transportJQuery) {
      var settings = $.extend({}, options);
      var deferred = $.Deferred();
      var job = {
        settings: settings,
        deferred: deferred,
        xhr: null,
        done: false,
        timer: null,
        transport: transportJQuery || $
      };
      var url = new URL(settings.url || location.href, location.href);
      var data = settings.data || {};
      var method = typeof data === "object" ? data.method || url.searchParams.get("method") : url.searchParams.get("method");
      job.priority = /^(POST|PUT|DELETE)$/i.test(settings.type || "") || method === "storage.set" || /_sns_.*(?:verify|edit_source)/.test(url.search) ? 0 : 1;
      job.background = settings.snsBackground === true;
      delete settings.snsBackground;
      if (!settings.timeout) settings.timeout = 1e4;
      var facade = deferred.promise({
        readyState: 0,
        status: 0,
        statusText: "queued"
      });
      function reflect(xhr) {
        if (!xhr) return;
        [ "readyState", "status", "statusText", "responseText", "responseJSON", "responseXML" ].forEach(function(key) {
          if (xhr[key] !== undefined) facade[key] = xhr[key];
        });
      }
      job.reflect = reflect;
      job.facade = facade;
      facade.abort = function(reason) {
        if (job.done) return facade;
        if (job.xhr) job.xhr.abort(reason || "abort"); else {
          job.done = true;
          clearTimeout(job.timer);
          facade.statusText = reason || "abort";
          deferred.rejectWith(settings.context || settings, [ facade, facade.statusText, facade.statusText ]);
          snsPump();
        }
        return facade;
      };
      facade.getResponseHeader = function(name) {
        return job.xhr ? job.xhr.getResponseHeader(name) : null;
      };
      facade.getAllResponseHeaders = function() {
        return job.xhr ? job.xhr.getAllResponseHeaders() : null;
      };
      job.timer = setTimeout(function() {
        if (!job.xhr) facade.abort("queue-timeout");
      }, 2e4);
      snsQueue.push(job);
      snsPump();
      return facade;
    }
    function snsPump() {
      if (snsPumpTimer) {
        clearTimeout(snsPumpTimer);
        snsPumpTimer = null;
      }
      snsQueue = snsQueue.filter(function(job) {
        return !job.done;
      });
      if (snsActive >= 2 || !snsQueue.length) return;
      var candidates = snsQueue.filter(function(job) {
        return !job.background || !document.hidden;
      });
      if (!candidates.length) return;
      candidates.sort(function(a, b) {
        return a.priority - b.priority;
      });
      var wait = Math.max(snsNextRequestAt, snsBlockedUntil) - Date.now();
      if (wait > 0) {
        snsPumpTimer = setTimeout(snsPump, wait);
        return;
      }
      var job = candidates[0];
      snsQueue.splice(snsQueue.indexOf(job), 1);
      clearTimeout(job.timer);
      snsActive++;
      snsStats.requests++;
      snsNextRequestAt = Date.now() + 1e3;
      job.facade.readyState = 1;
      try {
        job.xhr = job.transport.ajax(job.settings);
        job.xhr.done(function() {
          job.reflect(job.xhr);
          job.done = true;
          job.deferred.resolveWith(this, arguments);
        });
        job.xhr.fail(function(xhr, status, error) {
          job.reflect(xhr);
          if (xhr.status === 429 || xhr.status === 503) {
            var retry = xhr.getResponseHeader("Retry-After");
            var ms = retry ? /^\s*\d+\s*$/.test(retry) ? Number(retry) * 1e3 : Date.parse(retry) - Date.now() : 0;
            if (!isFinite(ms) || ms <= 0) ms = xhr.status === 429 ? 6e4 : 15e3;
            snsBlockedUntil = Math.max(snsBlockedUntil, Date.now() + ms);
            snsStats.rateLimits++;
          }
          job.done = true;
          job.deferred.rejectWith(this, [ job.facade, status, error ]);
        });
        job.xhr.always(function() {
          snsActive--;
          snsPump();
        });
      } catch (error) {
        snsActive--;
        job.done = true;
        job.deferred.rejectWith(job.settings, [ job.facade, "error", error ]);
      }
      snsPump();
    }
    document.addEventListener("visibilitychange", function() {
      if (!document.hidden) snsPump();
    });
    window.SNS_AUDIO_UPLOAD_CONFIG = window.SNS_AUDIO_UPLOAD_CONFIG || {
      cloudName: "oiywr18p",
      uploadPreset: "sns_audio",
      maxMb: 15
    };
    (function() {
      if (!document.getElementById("sns-chat-v03-style")) {
        var style = document.createElement("style");
        style.id = "sns-chat-v03-style";
        style.textContent = "/* =========================================================\n   1. \u041e\u0411\u0429\u0410\u042f \u041e\u0411\u041e\u041b\u041e\u0427\u041a\u0410 SNS\n========================================================= */\n\nbody.sns-chat-page #sns-chat-shell {\n    position: relative;\n    box-sizing: border-box;\n\n    width: 100%;\n    margin: 0;\n\n    overflow: hidden;\n\n    border-radius: 16px;\n}\n\n\n/* =========================================================\n   2. \u0428\u0410\u041f\u041a\u0410 \u0427\u0410\u0422\u0410\n========================================================= */\n\nbody.sns-chat-page #sns-chat-header {\n    box-sizing: border-box;\n\n    width: 100%;\n    min-height: 72px;\n\n    margin: 0;\n    padding: 13px 18px;\n\n    display: flex;\n    align-items: center;\n    justify-content: space-between;\n    gap: 15px;\n\n    background: #171717;\n    color: #f4f4f4;\n\n    border-radius: 16px 16px 0 0;\n}\n\n\nbody.sns-chat-page #sns-chat-header .sns-head-left {\n    min-width: 0;\n\n    display: flex;\n    align-items: center;\n    gap: 12px;\n}\n\n\nbody.sns-chat-page #sns-chat-header .sns-head-avatar {\n    display: block;\n\n    flex: 0 0 44px;\n\n    width: 44px;\n    height: 44px;\n\n    object-fit: cover;\n\n    border-radius: 50%;\n}\n\n\nbody.sns-chat-page #sns-chat-header .sns-head-avatar-empty {\n    display: flex;\n    align-items: center;\n    justify-content: center;\n\n    background: #333;\n    color: #eee;\n\n    font: 600 16px Arial, sans-serif;\n}\n\n\nbody.sns-chat-page #sns-chat-header .sns-head-text {\n    min-width: 0;\n}\n\n\nbody.sns-chat-page #sns-chat-header .sns-head-title {\n    overflow: hidden;\n\n    color: #f4f4f4;\n\n    font: 600 14px/1.3 Arial, sans-serif;\n\n    text-overflow: ellipsis;\n    white-space: nowrap;\n}\n\n\nbody.sns-chat-page #sns-chat-header .sns-head-subtitle {\n    margin-top: 4px;\n\n    color: #888;\n\n    font: 10px/1.2 Arial, sans-serif;\n}\n\n\nbody.sns-chat-page #sns-chat-header .sns-head-badge {\n    flex: 0 0 auto;\n\n    padding: 5px 9px;\n\n    color: #aaa;\n\n    border: 1px solid #444;\n    border-radius: 20px;\n\n    font: 600 8px/1 Arial, sans-serif;\n    letter-spacing: 1.7px;\n}\n\n\n/* =========================================================\n   3. \u041e\u0411\u041b\u0410\u0421\u0422\u042c \u0421\u041e\u041e\u0411\u0429\u0415\u041d\u0418\u0419\n========================================================= */\n\nbody.sns-chat-page #sns-chat-shell > .topic {\n    box-sizing: border-box;\n\n    width: 100%;\n\n    height: 540px;\n\n    margin: 0 !important;\n    padding: 26px 20px !important;\n\n    overflow-x: hidden;\n    overflow-y: auto;\n\n    background:\n        linear-gradient(\n            180deg,\n            #ecebe7 0%,\n            #f6f5f1 100%\n        ) !important;\n\n    border: 0 !important;\n    border-radius: 0 !important;\n}\n\n\n/* scrollbar */\n\nbody.sns-chat-page #sns-chat-shell > .topic::-webkit-scrollbar {\n    width: 6px;\n}\n\nbody.sns-chat-page #sns-chat-shell > .topic::-webkit-scrollbar-thumb {\n    background: rgba(0,0,0,.16);\n    border-radius: 10px;\n}\n\nbody.sns-chat-page #sns-chat-shell > .topic::-webkit-scrollbar-track {\n    background: transparent;\n}\n\n\n/* =========================================================\n   4. \u041f\u0415\u0420\u0412\u042b\u0419 \u041f\u041e\u0421\u0422 \u2014 \u0422\u0415\u0425\u041d\u0418\u0427\u0415\u0421\u041a\u0418\u0419\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-tech-post {\n    display: none !important;\n}\n\n\n/* =========================================================\n   5. \u0421\u041e\u041e\u0411\u0429\u0415\u041d\u0418\u042f\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-message {\n    position: relative;\n\n    clear: both;\n\n    margin: 0 0 13px !important;\n    padding: 0 !important;\n\n    background: transparent !important;\n\n    border: 0 !important;\n    box-shadow: none !important;\n}\n\n\nbody.sns-chat-page #pun-viewtopic .sns-message > .container {\n    position: relative;\n\n    box-sizing: border-box;\n\n    width: 100% !important;\n    min-height: 0 !important;\n\n    margin: 0 !important;\n    padding: 0 !important;\n\n    background: transparent !important;\n\n    border: 0 !important;\n    box-shadow: none !important;\n}\n\n\n/* =========================================================\n   6. \u0423\u0411\u0418\u0420\u0410\u0415\u041c \u0421\u0422\u0410\u041d\u0414\u0410\u0420\u0422\u041d\u042b\u0419 \u0412\u0418\u0414 \u041f\u041e\u0421\u0422\u0410\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-message .post-author,\nbody.sns-chat-page #pun-viewtopic .sns-message > h3,\nbody.sns-chat-page #pun-viewtopic .sns-message .container > h3,\nbody.sns-chat-page #pun-viewtopic .sns-message .post-body > h3,\nbody.sns-chat-page #pun-viewtopic .sns-message .post-links,\nbody.sns-chat-page #pun-viewtopic .sns-message .post-rating,\nbody.sns-chat-page #pun-viewtopic .sns-message .post-sig,\nbody.sns-chat-page #pun-viewtopic .sns-message .lastedit {\n    display: none !important;\n}\n\n\n/* =========================================================\n   7. \u0422\u0415\u041b\u041e \u0421\u041e\u041e\u0411\u0429\u0415\u041d\u0418\u042f\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-message .post-body {\n    position: relative;\n\n    float: none !important;\n\n    box-sizing: border-box;\n\n    width: fit-content !important;\n    max-width: 72% !important;\n    min-width: 68px;\n\n    padding: 0 !important;\n\n    background: transparent !important;\n\n    border: 0 !important;\n}\n\n\n/* \u0427\u0423\u0416\u041e\u0415 \u2014 \u0421\u041b\u0415\u0412\u0410 */\n\nbody.sns-chat-page #pun-viewtopic .sns-message.sns-other .post-body {\n    margin: 0 auto 0 42px !important;\n}\n\n\n/* \u0421\u0412\u041e\u0415 \u2014 \u0421\u041f\u0420\u0410\u0412\u0410 */\n\nbody.sns-chat-page #pun-viewtopic .sns-message.sns-own .post-body {\n    margin: 0 8px 0 auto !important;\n}\n\n\n/* \u0441\u0442\u0430\u043d\u0434\u0430\u0440\u0442\u043d\u0430\u044f \u0432\u043d\u0443\u0442\u0440\u0435\u043d\u043d\u044f\u044f \u043e\u0431\u0435\u0440\u0442\u043a\u0430 */\n\nbody.sns-chat-page #pun-viewtopic .sns-message .post-box {\n    width: auto !important;\n    min-height: 0 !important;\n\n    margin: 0 !important;\n    padding: 0 !important;\n\n    background: transparent !important;\n\n    border: 0 !important;\n}\n\n\n/* =========================================================\n   8. \u041f\u0423\u0417\u042b\u0420\u042c\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-message .post-content {\n    box-sizing: border-box;\n\n    width: auto !important;\n    min-height: 0 !important;\n\n    margin: 0 !important;\n    padding: 10px 13px !important;\n\n    text-align: left;\n\n    border: 0 !important;\n\n    font: 13px/1.45 Arial, sans-serif !important;\n\n    overflow-wrap: anywhere;\n}\n\n\n/* \u0447\u0443\u0436\u043e\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435 */\n\nbody.sns-chat-page #pun-viewtopic .sns-message.sns-other .post-content {\n    background: #fff !important;\n    color: #222 !important;\n\n    border-radius: 17px 17px 17px 5px;\n}\n\n\n/* \u043c\u043e\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435 */\n\nbody.sns-chat-page #pun-viewtopic .sns-message.sns-own .post-content {\n    background: #202020 !important;\n    color: #f5f5f5 !important;\n\n    border-radius: 17px 17px 5px 17px;\n}\n\n\nbody.sns-chat-page #pun-viewtopic .sns-message .post-content p {\n    margin: 0 0 7px !important;\n}\n\n\nbody.sns-chat-page #pun-viewtopic .sns-message .post-content p:last-child {\n    margin-bottom: 0 !important;\n}\n\n\n/* =========================================================\n   9. \u041a\u0410\u0420\u0422\u0418\u041d\u041a\u0418 \u0412 \u0421\u041e\u041e\u0411\u0429\u0415\u041d\u0418\u0418\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-message .post-content img {\n    display: block;\n\n    max-width: min(100%, 430px) !important;\n    height: auto !important;\n\n    margin: 3px 0 !important;\n\n    border-radius: 12px;\n}\n\n\n/* \u0435\u0441\u043b\u0438 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435 \u0441\u043e\u0441\u0442\u043e\u0438\u0442 \u043f\u0440\u0430\u043a\u0442\u0438\u0447\u0435\u0441\u043a\u0438 \u0442\u043e\u043b\u044c\u043a\u043e \u0438\u0437 \u043a\u0430\u0440\u0442\u0438\u043d\u043a\u0438 */\n\nbody.sns-chat-page #pun-viewtopic .sns-message .post-content a img {\n    margin: 0 !important;\n}\n\n\n/* =========================================================\n   10. \u041c\u0410\u041b\u0415\u041d\u042c\u041a\u0418\u0419 \u0410\u0412\u0410\u0422\u0410\u0420 \u0421\u041e\u0411\u0415\u0421\u0415\u0414\u041d\u0418\u041a\u0410\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-mini-avatar {\n    position: absolute;\n\n    left: 3px;\n    bottom: 0;\n\n    display: block;\n\n    width: 30px;\n    height: 30px;\n\n    object-fit: cover;\n\n    border-radius: 50%;\n}\n\n\nbody.sns-chat-page #pun-viewtopic .sns-mini-avatar-empty {\n    display: flex;\n    align-items: center;\n    justify-content: center;\n\n    background: #d5d3ce;\n    color: #666;\n\n    font: 600 11px Arial, sans-serif;\n}\n\n\n/* =========================================================\n   11. \u0410\u0412\u0422\u041e\u0420 + \u0412\u0420\u0415\u041c\u042f\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-meta {\n    box-sizing: border-box;\n\n    display: flex;\n    align-items: center;\n    gap: 7px;\n\n    min-height: 15px;\n\n    margin: 0 4px 4px;\n\n    color: #999;\n\n    font: 9px/1.2 Arial, sans-serif;\n}\n\n\nbody.sns-chat-page #pun-viewtopic .sns-own .sns-meta {\n    justify-content: flex-end;\n}\n\n\nbody.sns-chat-page #pun-viewtopic .sns-author-name {\n    color: #666;\n    font-weight: 600;\n}\n\n\n/* =========================================================\n   12. \u041c\u0415\u041d\u042e \u0421\u041e\u041e\u0411\u0429\u0415\u041d\u0418\u042f\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-controls {\n    position: absolute;\n\n    top: 50%;\n    z-index: 40;\n\n    transform: translateY(-50%);\n\n    opacity: .35;\n\n    transition: opacity .15s ease;\n}\n\n\nbody.sns-chat-page #pun-viewtopic .sns-message:hover .sns-controls {\n    opacity: 1;\n}\n\n\n/* \u0443 \u0441\u0432\u043e\u0435\u0433\u043e \u2014 \u0441\u043b\u0435\u0432\u0430 \u043e\u0442 \u043f\u0443\u0437\u044b\u0440\u044f */\n\nbody.sns-chat-page #pun-viewtopic .sns-own .sns-controls {\n    left: -31px;\n}\n\n\n/* \u0443 \u0447\u0443\u0436\u043e\u0433\u043e \u2014 \u0441\u043f\u0440\u0430\u0432\u0430 */\n\nbody.sns-chat-page #pun-viewtopic .sns-other .sns-controls {\n    right: -31px;\n}\n\n\nbody.sns-chat-page .sns-menu-toggle {\n    box-sizing: border-box;\n\n    width: 26px;\n    height: 26px;\n\n    margin: 0;\n    padding: 0;\n\n    cursor: pointer;\n\n    background: transparent;\n    color: #777;\n\n    border: 0;\n    border-radius: 50%;\n\n    font: 700 11px/26px Arial, sans-serif;\n    letter-spacing: -1px;\n}\n\n\nbody.sns-chat-page .sns-menu-toggle:hover {\n    background: rgba(0,0,0,.06);\n    color: #222;\n}\n\n\nbody.sns-chat-page .sns-menu {\n    position: absolute;\n\n    top: 28px;\n    z-index: 200;\n\n    display: none;\n\n    min-width: 140px;\n\n    padding: 5px;\n\n    background: #fff;\n\n    border: 1px solid rgba(0,0,0,.09);\n    border-radius: 10px;\n\n    box-shadow: 0 7px 24px rgba(0,0,0,.14);\n}\n\n\nbody.sns-chat-page .sns-own .sns-menu {\n    right: 0;\n}\n\n\nbody.sns-chat-page .sns-other .sns-menu {\n    left: 0;\n}\n\n\nbody.sns-chat-page .sns-menu.is-open {\n    display: block;\n}\n\n\nbody.sns-chat-page .sns-menu button {\n    display: block;\n\n    box-sizing: border-box;\n\n    width: 100%;\n\n    padding: 8px 10px;\n\n    cursor: pointer;\n\n    background: transparent;\n    color: #333;\n\n    border: 0;\n    border-radius: 7px;\n\n    font: 11px Arial, sans-serif;\n    text-align: left;\n}\n\n\nbody.sns-chat-page .sns-menu button:hover {\n    background: #f1f1f1;\n}\n\n\n/* =========================================================\n   13. \u041f\u0423\u0421\u0422\u041e\u0419 \u0427\u0410\u0422\n========================================================= */\n\nbody.sns-chat-page #sns-empty {\n    box-sizing: border-box;\n\n    min-height: 100%;\n\n    display: flex;\n    flex-direction: column;\n    align-items: center;\n    justify-content: center;\n\n    padding: 30px;\n\n    color: #999;\n\n    font: 11px/1.5 Arial, sans-serif;\n    text-align: center;\n}\n\n\nbody.sns-chat-page #sns-empty b {\n    margin-bottom: 5px;\n\n    color: #555;\n\n    font-size: 13px;\n}\n\n\n/* =========================================================\n   14. \u041d\u0410\u0428\u0410 \u0421\u0422\u0420\u041e\u041a\u0410 \u0412\u0412\u041e\u0414\u0410\n========================================================= */\n\nbody.sns-chat-page #sns-composer-ui {\n    box-sizing: border-box;\n\n    display: flex;\n    align-items: flex-end;\n    gap: 9px;\n\n    width: 100%;\n\n    margin: 0;\n    padding: 11px 13px;\n\n    background: #171717;\n\n    border-top: 1px solid #292929;\n    border-radius: 0 0 16px 16px;\n}\n\n\n/* =========================================================\n   15. \u041f\u041b\u042e\u0421\n========================================================= */\n\nbody.sns-chat-page .sns-ui-plus {\n    flex: 0 0 40px;\n\n    box-sizing: border-box;\n\n    width: 40px;\n    height: 40px;\n\n    margin: 0;\n    padding: 0;\n\n    cursor: pointer;\n\n    background: #292929;\n    color: #aaa;\n\n    border: 0;\n    border-radius: 50%;\n\n    font: 300 23px/40px Arial, sans-serif;\n\n    transition:\n        background .15s ease,\n        color .15s ease,\n        transform .15s ease;\n}\n\n\nbody.sns-chat-page .sns-ui-plus:hover {\n    background: #333;\n    color: #fff;\n}\n\n\nbody.sns-chat-page .sns-ui-plus.is-active {\n    transform: rotate(45deg);\n}\n\n\n/* =========================================================\n   16. \u041f\u041e\u041b\u0415 \u0421\u041e\u041e\u0411\u0429\u0415\u041d\u0418\u042f\n========================================================= */\n\nbody.sns-chat-page .sns-ui-input-wrap {\n    flex: 1 1 auto;\n\n    min-width: 0;\n}\n\n\nbody.sns-chat-page #sns-ui-input {\n    display: block;\n\n    box-sizing: border-box;\n\n    width: 100%;\n    height: 40px;\n    min-height: 40px;\n    max-height: 120px;\n\n    margin: 0;\n    padding: 10px 15px;\n\n    resize: none;\n\n    overflow-y: auto;\n\n    background: #252525;\n    color: #f3f3f3;\n\n    border: 1px solid #333;\n    border-radius: 20px;\n\n    outline: none;\n\n    font: 13px/20px Arial, sans-serif;\n}\n\n\nbody.sns-chat-page #sns-ui-input:focus {\n    border-color: #555;\n}\n\n\nbody.sns-chat-page #sns-ui-input::placeholder {\n    color: #777;\n}\n\n\n/* =========================================================\n   17. \u041a\u041d\u041e\u041f\u041a\u0410 \u041e\u0422\u041f\u0420\u0410\u0412\u041a\u0418\n========================================================= */\n\nbody.sns-chat-page .sns-ui-send {\n    flex: 0 0 40px;\n\n    box-sizing: border-box;\n\n    width: 40px;\n    height: 40px;\n\n    margin: 0;\n    padding: 0;\n\n    cursor: pointer;\n\n    background: #f3f2ef;\n    color: #171717;\n\n    border: 0;\n    border-radius: 50%;\n\n    font: 600 14px/40px Arial, sans-serif;\n\n    transition:\n        transform .15s ease,\n        opacity .15s ease;\n}\n\n\nbody.sns-chat-page .sns-ui-send:hover {\n    transform: scale(1.04);\n}\n\n\nbody.sns-chat-page .sns-ui-send:active {\n    transform: scale(.96);\n}\n\n\nbody.sns-chat-page .sns-ui-send:disabled {\n    cursor: default;\n\n    opacity: .4;\n\n    transform: none;\n}\n\n\n/* =========================================================\n   18. \u041c\u0415\u041d\u042e +\n========================================================= */\n\nbody.sns-chat-page #sns-attach-menu {\n    position: absolute;\n\n    left: 12px;\n    bottom: 64px;\n\n    z-index: 300;\n\n    display: none;\n\n    min-width: 145px;\n\n    padding: 6px;\n\n    background: #222;\n    color: #eee;\n\n    border: 1px solid #333;\n    border-radius: 11px;\n\n    box-shadow: 0 9px 30px rgba(0,0,0,.18);\n}\n\n\nbody.sns-chat-page #sns-attach-menu.is-open {\n    display: block;\n}\n\n\nbody.sns-chat-page #sns-attach-menu button {\n    display: flex;\n    align-items: center;\n    gap: 8px;\n\n    box-sizing: border-box;\n\n    width: 100%;\n\n    margin: 0;\n    padding: 8px 10px;\n\n    cursor: pointer;\n\n    background: transparent;\n    color: #ddd;\n\n    border: 0;\n    border-radius: 7px;\n\n    font: 11px Arial, sans-serif;\n    text-align: left;\n}\n\n\nbody.sns-chat-page #sns-attach-menu button:hover {\n    background: #303030;\n    color: #fff;\n}\n\nbody.sns-chat-page #sns-attach-menu .sns-attach-icon {\n    display: inline-flex;\n    align-items: center;\n    justify-content: center;\n    flex: 0 0 16px;\n\n    width: 16px;\n    height: 16px;\n\n    opacity: .88;\n}\n\nbody.sns-chat-page #sns-attach-menu .sns-attach-icon svg {\n    display: block;\n\n    width: 16px;\n    height: 16px;\n\n    fill: none;\n    stroke: currentColor;\n    stroke-width: 2;\n    stroke-linecap: round;\n    stroke-linejoin: round;\n}\n\nbody.sns-chat-page #sns-attach-menu .sns-attach-icon-photo svg {\n    width: 17px;\n    height: 17px;\n    fill: currentColor;\n    stroke: none;\n}\n\nbody.sns-chat-page #sns-attach-menu .sns-attach-label {\n    display: block;\n    line-height: 1.2;\n}\n\n\n/* =========================================================\n   19. \u0420\u041e\u0414\u041d\u0410\u042f \u0424\u041e\u0420\u041c\u0410 RUSFF\n\n   \u041e\u043d\u0430 \u0440\u0430\u0431\u043e\u0442\u0430\u0435\u0442, \u043d\u043e \u0438\u0433\u0440\u043e\u043a \u0435\u0435 \u043d\u0435 \u0432\u0438\u0434\u0438\u0442.\n========================================================= */\n\nbody.sns-chat-page #post.sns-native-form {\n    position: fixed !important;\n\n    left: -10000px !important;\n    top: -10000px !important;\n\n    z-index: -100 !important;\n\n    display: block !important;\n\n    width: 1px !important;\n    height: 1px !important;\n\n    margin: 0 !important;\n    padding: 0 !important;\n\n    overflow: hidden !important;\n\n    opacity: 0 !important;\n\n    pointer-events: none !important;\n}\n\n\n/* =========================================================\n   20. \u0411\u042b\u0421\u0422\u0420\u041e\u0415 \u0420\u0415\u0414\u0410\u041a\u0422\u0418\u0420\u041e\u0412\u0410\u041d\u0418\u0415\n========================================================= */\n\nbody.sns-chat-page #pun-viewtopic .sns-message textarea {\n    box-sizing: border-box !important;\n\n    width: 100% !important;\n    min-width: 250px;\n    min-height: 85px !important;\n\n    padding: 10px !important;\n\n    background: #fff !important;\n    color: #222 !important;\n\n    border: 1px solid #ddd !important;\n    border-radius: 10px !important;\n\n    outline: none;\n\n    font: 12px/1.45 Arial, sans-serif !important;\n}\n\n\n/* =========================================================\n   21. \u041c\u041e\u0411\u0418\u041b\u042c\u041d\u0410\u042f \u0412\u0415\u0420\u0421\u0418\u042f\n========================================================= */\n\n@media (max-width: 650px) {\n\n    body.sns-chat-page #sns-chat-shell > .topic {\n        height: 65vh;\n\n        padding: 18px 10px !important;\n    }\n\n\n    body.sns-chat-page #pun-viewtopic .sns-message .post-body {\n        max-width: 82% !important;\n    }\n\n\n    body.sns-chat-page #sns-composer-ui {\n        padding: 9px;\n        gap: 7px;\n    }\n\n\n    body.sns-chat-page .sns-ui-plus,\n    body.sns-chat-page .sns-ui-send {\n        flex-basis: 38px;\n\n        width: 38px;\n        height: 38px;\n\n        line-height: 38px;\n    }\n\n\n    body.sns-chat-page #sns-ui-input {\n        height: 38px;\n        min-height: 38px;\n\n        padding-top: 9px;\n        padding-bottom: 9px;\n    }\n\n}";
        document.head.appendChild(style);
      }
    })();
    (function() {
      if (!document.getElementById("sns-raw-post-antiflash-style")) {
        var antiFlashStyle = document.createElement("style");
        antiFlashStyle.id = "sns-raw-post-antiflash-style";
        antiFlashStyle.textContent = [ "body.sns-chat-page #pun-viewtopic .topic .post:not(.sns-message){", "display:none!important;", "visibility:hidden!important;", "opacity:0!important;", "pointer-events:none!important;", "}", "body.sns-chat-page #pun-viewtopic .topic .post.sns-message{", "visibility:visible!important;", "opacity:1!important;", "pointer-events:auto!important;", "}", "body.sns-chat-page #pun-viewtopic .topic .post.sns-tech-post,", "body.sns-chat-page #pun-viewtopic .topic .post.sns-config-post{", "display:none!important;", "visibility:hidden!important;", "opacity:0!important;", "}" ].join("");
        document.head.appendChild(antiFlashStyle);
      }
    })();
    (function() {
      if (!document.getElementById("sns-chat-photo-style")) {
        var photoStyle = document.createElement("style");
        photoStyle.id = "sns-chat-photo-style";
        photoStyle.textContent = [ "body.sns-chat-page #sns-chat-shell #image-area.sns-image-area {", "position:absolute!important;", "left:12px!important;", "right:12px!important;", "bottom:64px!important;", "top:auto!important;", "z-index:500!important;", "box-sizing:border-box!important;", "width:auto!important;", "max-width:none!important;", "max-height:420px!important;", "margin:0!important;", "padding:12px!important;", "overflow:auto!important;", "background:#f4f3ef!important;", "color:#222!important;", "border:1px solid rgba(0,0,0,.14)!important;", "border-radius:13px!important;", "box-shadow:0 14px 38px rgba(0,0,0,.24)!important;", "}", "body.sns-chat-page #sns-chat-shell #image-area.sns-image-area a{cursor:pointer;}", "body.sns-chat-page #sns-chat-shell #image-area.sns-image-area input,", "body.sns-chat-page #sns-chat-shell #image-area.sns-image-area select,", "body.sns-chat-page #sns-chat-shell #image-area.sns-image-area button{max-width:100%;}", "body.sns-chat-page #sns-chat-shell #image-area.sns-image-area img{max-width:100%;height:auto;}", "body.sns-chat-page #sns-chat-shell #image-area.sns-image-area{padding-top:34px!important;}", "body.sns-chat-page #sns-chat-shell #image-area .sns-image-area-close{", "position:absolute!important;right:9px!important;top:7px!important;z-index:30!important;", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:25px!important;height:25px!important;margin:0!important;padding:0!important;", "border:0!important;border-radius:50%!important;cursor:pointer!important;", "background:rgba(0,0,0,.08)!important;color:#555!important;", "font:20px/1 Arial,sans-serif!important;", "}" ].join("");
        document.head.appendChild(photoStyle);
      }
    })();
    (function() {
      if (!document.getElementById("sns-voice-message-style")) {
        var voiceStyle = document.createElement("style");
        voiceStyle.id = "sns-voice-message-style";
        voiceStyle.textContent = [ "body.sns-chat-page #pun-viewtopic .sns-message.sns-voice-message .post-body{width:245px!important;min-width:245px!important;max-width:245px!important;}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-voice-message .post-content{padding:8px 10px!important;overflow:visible!important;}", "body.sns-chat-page .sns-voice-card{display:block;width:225px!important;min-width:225px!important;max-width:225px!important;box-sizing:border-box;color:inherit;}", "body.sns-chat-page .sns-voice-toggle{", "display:grid;grid-template-columns:32px minmax(0,1fr) auto;align-items:center;column-gap:7px;", "width:100%;min-width:0;max-width:100%;margin:0;padding:0;border:0;background:transparent;color:inherit;cursor:pointer;", "font:inherit;text-align:left;box-sizing:border-box;", "}", "body.sns-chat-page .sns-voice-play{", "display:flex;align-items:center;justify-content:center;width:32px;height:32px;border-radius:50%;", "background:rgba(255,255,255,.18);box-shadow:inset 0 0 0 1px rgba(255,255,255,.08);", "flex:0 0 32px;box-sizing:border-box;", "}", "body.sns-chat-page .sns-other .sns-voice-play{background:rgba(0,0,0,.07);box-shadow:inset 0 0 0 1px rgba(0,0,0,.04);}", "body.sns-chat-page .sns-voice-play svg{display:block;width:13px;height:13px;fill:currentColor;stroke:none;}", "body.sns-chat-page .sns-voice-play .sns-voice-play-icon{transform:translateX(1px);}", "body.sns-chat-page .sns-voice-play .sns-voice-pause-icon{display:none;transform:none;}", "body.sns-chat-page .sns-voice-card.is-open .sns-voice-play .sns-voice-play-icon{display:none;}", "body.sns-chat-page .sns-voice-card.is-open .sns-voice-play .sns-voice-pause-icon{display:block;}", "body.sns-chat-page .sns-voice-wave{display:flex;align-items:center;justify-content:space-between;gap:1px;height:28px;min-width:0;overflow:hidden;}", "body.sns-chat-page .sns-voice-wave i{display:block;flex:0 0 2px;width:2px;min-width:2px;max-width:2px;height:var(--sns-vh,8px);border-radius:3px;background:currentColor;opacity:.70;}", "body.sns-chat-page .sns-voice-time{font:700 9px/1 Arial,sans-serif;opacity:.76;white-space:nowrap;}", "body.sns-chat-page .sns-voice-transcript{", "display:none;width:100%;max-width:100%;box-sizing:border-box;margin:8px 0 1px;padding:8px 4px 2px;border-top:1px solid currentColor;", "font:12px/1.45 Arial,sans-serif;white-space:pre-wrap;overflow-wrap:anywhere;word-break:break-word;opacity:.92;", "border-top-color:rgba(255,255,255,.22);", "}", "body.sns-chat-page .sns-other .sns-voice-transcript{border-top-color:rgba(0,0,0,.10);}", "body.sns-chat-page .sns-voice-card.is-open .sns-voice-transcript{display:block;}", "body.sns-chat-page #sns-voice-compose{", "position:absolute;left:64px;right:58px;bottom:64px;z-index:2700;display:none;", "box-sizing:border-box;padding:11px;background:rgba(255,255,255,.985);", "border:1px solid rgba(0,0,0,.07);border-radius:12px;", "box-shadow:0 10px 28px rgba(18,20,28,.12),0 22px 52px rgba(18,20,28,.13);", "}", "body.sns-chat-page #sns-voice-compose.is-open{display:block;}", "body.sns-chat-page .sns-voice-compose-title{display:flex;align-items:center;gap:7px;margin:0 0 8px;color:#444;font:700 11px/1.2 Arial,sans-serif;}", "body.sns-chat-page .sns-voice-compose-title svg{width:14px;height:14px;fill:none;stroke:var(--sns-own-g1,#b65f3a);stroke-width:2;stroke-linecap:round;stroke-linejoin:round;}", "body.sns-chat-page #sns-voice-compose-text{", "display:block;box-sizing:border-box;width:100%;min-height:78px;max-height:180px;resize:vertical;", "padding:10px 11px;background:#f7f7f7;color:#333;border:1px solid rgba(0,0,0,.08);border-radius:8px;", "outline:none;font:12px/1.45 Arial,sans-serif;", "}", "body.sns-chat-page #sns-voice-compose-text:focus{border-color:color-mix(in srgb,var(--sns-own-g1,#b65f3a) 42%,transparent);}", "body.sns-chat-page .sns-voice-compose-foot{display:flex;align-items:center;justify-content:space-between;gap:10px;margin-top:8px;}", "body.sns-chat-page .sns-voice-compose-hint{min-width:0;color:#999;font:9px/1.25 Arial,sans-serif;}", "body.sns-chat-page .sns-voice-compose-actions{display:flex;align-items:center;gap:5px;flex:0 0 auto;}", "body.sns-chat-page .sns-voice-compose-actions button{", "height:29px;margin:0;padding:0 10px;border:0;border-radius:7px;cursor:pointer;", "font:700 9px/29px Arial,sans-serif;", "}", "body.sns-chat-page .sns-voice-cancel{background:transparent;color:#777;}", "body.sns-chat-page .sns-voice-submit{background:var(--sns-own-g1,#b65f3a);color:#fff;}", "body.sns-chat-page .sns-voice-submit:disabled{opacity:.35;cursor:default;}", "body.sns-chat-page #sns-attach-menu .sns-attach-voice:before{content:none!important;display:none!important;}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.sns-pending-voice .sns-pending-bubble{padding:8px 10px!important;}", "body.sns-chat-page #sns-pending-stack .sns-pending-voice .sns-voice-card{min-width:215px;}", "body.sns-chat-page #sns-pending-stack .sns-pending-voice .sns-voice-transcript{display:none!important;}", "@media(max-width:650px){", "body.sns-chat-page #pun-viewtopic .sns-message.sns-voice-message .post-body{width:220px!important;min-width:220px!important;max-width:220px!important;}", "body.sns-chat-page .sns-voice-card{width:200px!important;min-width:200px!important;max-width:200px!important;}", "body.sns-chat-page #sns-voice-compose{left:8px;right:8px;bottom:61px;}", "body.sns-chat-page .sns-voice-compose-foot{align-items:flex-end;}", "body.sns-chat-page .sns-voice-compose-hint{max-width:150px;}", "}" ].join("");
        document.head.appendChild(voiceStyle);
      }
    })();
    (function() {
      if (!document.getElementById("sns-audio-message-style")) {
        var audioStyle = document.createElement("style");
        audioStyle.id = "sns-audio-message-style";
        audioStyle.textContent = [ "body.sns-chat-page #pun-viewtopic .sns-message.sns-audio-message .post-body{width:315px!important;min-width:315px!important;max-width:315px!important;}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-audio-message .post-content{padding:10px!important;overflow:visible!important;}", "body.sns-chat-page .sns-audio-card{display:grid;grid-template-columns:48px minmax(0,1fr);gap:10px;width:295px;min-width:295px;max-width:295px;box-sizing:border-box;color:inherit;}", "body.sns-chat-page .sns-audio-cover{position:relative;display:flex;align-items:center;justify-content:center;width:48px;height:48px;overflow:hidden;border-radius:10px;background:rgba(255,255,255,.16);box-shadow:inset 0 0 0 1px rgba(255,255,255,.08);}", "body.sns-chat-page .sns-other .sns-audio-cover{background:rgba(0,0,0,.07);box-shadow:inset 0 0 0 1px rgba(0,0,0,.05);}", "body.sns-chat-page .sns-audio-cover img{display:block!important;width:100%!important;height:100%!important;max-width:none!important;margin:0!important;object-fit:cover!important;border-radius:0!important;}", "body.sns-chat-page .sns-audio-cover svg{display:block;width:22px;height:22px;fill:none;stroke:currentColor;stroke-width:1.8;stroke-linecap:round;stroke-linejoin:round;opacity:.78;}", "body.sns-chat-page .sns-audio-main{min-width:0;display:flex;flex-direction:column;justify-content:center;}", "body.sns-chat-page .sns-audio-title{display:block;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;font:700 12px/1.25 Arial,sans-serif;}", "body.sns-chat-page .sns-audio-artist{display:block;min-height:12px;margin-top:2px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;font:9px/1.2 Arial,sans-serif;opacity:.72;}", "body.sns-chat-page .sns-audio-controls{display:grid;grid-template-columns:25px minmax(0,1fr) auto;align-items:center;gap:7px;margin-top:7px;}", "body.sns-chat-page .sns-audio-play{display:flex;align-items:center;justify-content:center;width:25px;height:25px;margin:0;padding:0;border:0;border-radius:50%;background:rgba(255,255,255,.18);color:inherit;cursor:pointer;}", "body.sns-chat-page .sns-other .sns-audio-play{background:rgba(0,0,0,.07);}", "body.sns-chat-page .sns-audio-play svg{display:block;width:11px;height:11px;fill:currentColor;stroke:none;}", "body.sns-chat-page .sns-audio-play .sns-audio-pause-icon{display:none;}", "body.sns-chat-page .sns-audio-play.is-playing .sns-audio-play-icon{display:none;}", "body.sns-chat-page .sns-audio-play.is-playing .sns-audio-pause-icon{display:block;}", "body.sns-chat-page .sns-audio-track{position:relative;height:14px;cursor:pointer;}", 'body.sns-chat-page .sns-audio-track:before{content:"";position:absolute;left:0;right:0;top:6px;height:2px;border-radius:99px;background:currentColor;opacity:.22;}', "body.sns-chat-page .sns-audio-progress{position:absolute;left:0;top:6px;width:0;height:2px;border-radius:99px;background:currentColor;opacity:.9;pointer-events:none;}", "body.sns-chat-page .sns-audio-time{font:700 8px/1 Arial,sans-serif;white-space:nowrap;opacity:.72;}", "body.sns-chat-page .sns-audio-native{display:none!important;}", "body.sns-chat-page .sns-audio-error{display:none;grid-column:1/-1;margin-top:2px;font:8px/1.25 Arial,sans-serif;opacity:.65;}", "body.sns-chat-page .sns-audio-card.has-error .sns-audio-error{display:block;}", "body.sns-chat-page #sns-audio-compose{position:absolute;left:64px;right:58px;bottom:64px;z-index:2710;display:none;box-sizing:border-box;padding:11px;background:rgba(255,255,255,.985);border:1px solid rgba(0,0,0,.07);border-radius:12px;box-shadow:0 10px 28px rgba(18,20,28,.12),0 22px 52px rgba(18,20,28,.13);}", "body.sns-chat-page #sns-audio-compose.is-open{display:block;}", "body.sns-chat-page .sns-audio-compose-title{display:flex;align-items:center;gap:7px;margin:0 0 9px;color:#444;font:700 11px/1.2 Arial,sans-serif;}", "body.sns-chat-page .sns-audio-compose-title svg{width:15px;height:15px;fill:none;stroke:var(--sns-own-g1,#b65f3a);stroke-width:2;stroke-linecap:round;stroke-linejoin:round;}", "body.sns-chat-page .sns-audio-upload-box{display:grid;grid-template-columns:auto minmax(0,1fr);align-items:center;gap:9px;margin:0 0 9px;padding:8px 9px;background:#f6f6f7;border:1px solid rgba(0,0,0,.07);border-radius:9px;}", "body.sns-chat-page .sns-audio-file-input{display:none!important;}", "body.sns-chat-page .sns-audio-file-pick{height:32px;margin:0;padding:0 11px;border:0;border-radius:8px;cursor:pointer;background:#2c2c31;color:#fff;font:700 9px/32px Arial,sans-serif;white-space:nowrap;}", "body.sns-chat-page .sns-audio-file-pick:disabled{cursor:default;opacity:.38;}", "body.sns-chat-page .sns-audio-upload-info{display:flex;flex-direction:column;gap:5px;min-width:0;}", "body.sns-chat-page .sns-audio-upload-status{display:block;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;color:#888;font:9px/1.2 Arial,sans-serif;}", "body.sns-chat-page .sns-audio-upload-status.is-error{color:#b64e42;}", "body.sns-chat-page .sns-audio-upload-status.is-success{color:#62815c;}", "body.sns-chat-page .sns-audio-upload-progress{display:none;position:relative;width:100%;height:3px;overflow:hidden;background:rgba(0,0,0,.08);border-radius:99px;}", "body.sns-chat-page .sns-audio-upload-progress.is-visible{display:block;}", "body.sns-chat-page .sns-audio-upload-progress>i{display:block;width:0;height:100%;background:var(--sns-own-g1,#b65f3a);border-radius:inherit;transition:width .12s linear;}", "body.sns-chat-page .sns-audio-fields{display:grid;grid-template-columns:1fr 1fr;gap:7px;}", "body.sns-chat-page .sns-audio-field{display:flex;flex-direction:column;gap:4px;min-width:0;}", "body.sns-chat-page .sns-audio-field.is-wide{grid-column:1/-1;}", "body.sns-chat-page .sns-audio-field label{color:#888;font:700 8px/1.2 Arial,sans-serif;}", "body.sns-chat-page .sns-audio-field input{box-sizing:border-box;width:100%;height:34px;margin:0;padding:0 9px;background:#f7f7f7;color:#333;border:1px solid rgba(0,0,0,.08);border-radius:8px;outline:none;font:11px/34px Arial,sans-serif;}", "body.sns-chat-page .sns-audio-field input:focus{border-color:color-mix(in srgb,var(--sns-own-g1,#b65f3a) 42%,transparent);}", "body.sns-chat-page .sns-audio-field input.is-invalid{border-color:#c85b4a!important;}", "body.sns-chat-page .sns-audio-compose-foot{display:flex;align-items:center;justify-content:space-between;gap:10px;margin-top:9px;}", "body.sns-chat-page .sns-audio-compose-hint{min-width:0;color:#999;font:8px/1.3 Arial,sans-serif;}", "body.sns-chat-page .sns-audio-compose-actions{display:flex;align-items:center;gap:5px;flex:0 0 auto;}", "body.sns-chat-page .sns-audio-compose-actions button{height:29px;margin:0;padding:0 10px;border:0;border-radius:7px;cursor:pointer;font:700 9px/29px Arial,sans-serif;}", "body.sns-chat-page .sns-audio-cancel{background:transparent;color:#777;}", "body.sns-chat-page .sns-audio-submit{background:var(--sns-own-g1,#b65f3a);color:#fff;}", "body.sns-chat-page .sns-audio-submit:disabled{opacity:.35;cursor:default;}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.sns-pending-audio .sns-pending-bubble{padding:10px!important;}", "body.sns-chat-page #sns-pending-stack .sns-pending-audio .sns-audio-card{width:280px;min-width:280px;max-width:280px;}", "@media(max-width:650px){", "body.sns-chat-page #pun-viewtopic .sns-message.sns-audio-message .post-body{width:275px!important;min-width:275px!important;max-width:275px!important;}", "body.sns-chat-page .sns-audio-card{width:255px;min-width:255px;max-width:255px;grid-template-columns:44px minmax(0,1fr);gap:8px;}", "body.sns-chat-page .sns-audio-cover{width:44px;height:44px;}", "body.sns-chat-page #sns-audio-compose{left:8px;right:8px;bottom:61px;}", "body.sns-chat-page .sns-audio-upload-box{grid-template-columns:1fr;gap:6px;}", "body.sns-chat-page .sns-audio-file-pick{width:100%;}", "body.sns-chat-page .sns-audio-fields{grid-template-columns:1fr;}", "body.sns-chat-page .sns-audio-field.is-wide{grid-column:auto;}", "body.sns-chat-page .sns-audio-compose-foot{align-items:flex-end;}", "body.sns-chat-page .sns-audio-compose-hint{max-width:160px;}", "}" ].join("");
        document.head.appendChild(audioStyle);
      }
    })();
    (function() {
      if (!document.getElementById("sns-message-replies-style")) {
        var replyStyle = document.createElement("style");
        replyStyle.id = "sns-message-replies-style";
        replyStyle.textContent = [ "body.sns-chat-page .sns-reply-preview{display:block;width:100%;max-width:100%;box-sizing:border-box;margin:0 0 1px;padding:6px 8px;border:0;border-left:2px solid currentColor;border-radius:5px;background:rgba(255,255,255,.11);color:inherit;cursor:pointer;text-align:left;font:inherit;overflow:hidden;}", "body.sns-chat-page .sns-other .sns-reply-preview{background:rgba(0,0,0,.055);}", "body.sns-chat-page .sns-reply-preview-author{display:block;margin:0 0 2px;font:700 9px/1.2 Arial,sans-serif;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;}", "body.sns-chat-page .sns-reply-preview-text{display:block;font:10px/1.3 Arial,sans-serif;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;opacity:.76;}", "body.sns-chat-page .sns-reply-preview:hover{background:rgba(255,255,255,.17);}", "body.sns-chat-page .sns-other .sns-reply-preview:hover{background:rgba(0,0,0,.085);}", "body.sns-chat-page .sns-voice-message .sns-reply-preview,body.sns-chat-page .sns-audio-message .sns-reply-preview{margin-bottom:2px;}", "body.sns-chat-page .sns-reply-target-flash .post-content{animation:snsReplyFlash .9s ease!important;}", "@keyframes snsReplyFlash{0%,100%{filter:none;}35%{filter:brightness(1.18);box-shadow:0 0 0 3px rgba(255,255,255,.48)!important;}}", "body.sns-chat-page #sns-reply-compose{display:none!important;grid-column:1/-1!important;grid-row:1!important;align-items:center;gap:8px;box-sizing:border-box;width:100%;min-width:0;margin:0 0 7px;padding:6px 8px;background:rgba(255,255,255,.62);border-left:2px solid var(--sns-own-g1,#b65f3a);border-radius:6px;}", "body.sns-chat-page #sns-reply-compose.is-open{display:flex!important;}", "body.sns-chat-page .sns-reply-compose-copy{display:block;flex:1 1 auto;min-width:0;}", "body.sns-chat-page .sns-reply-compose-author{display:block;color:var(--sns-own-g1,#b65f3a);font:700 9px/1.2 Arial,sans-serif;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;}", "body.sns-chat-page .sns-reply-compose-text{display:block;margin-top:2px;color:#666;font:9px/1.25 Arial,sans-serif;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;}", "body.sns-chat-page .sns-reply-compose-close{display:flex;align-items:center;justify-content:center;flex:0 0 22px;width:22px;height:22px;margin:0;padding:0;border:0;border-radius:50%;background:transparent;color:#888;cursor:pointer;font:16px/22px Arial,sans-serif;}", "body.sns-chat-page .sns-reply-compose-close:hover{background:rgba(0,0,0,.06);color:#444;}", "body.sns-chat-page #sns-composer-ui{grid-template-rows:auto auto!important;}", "body.sns-chat-page #sns-composer-ui>.sns-ui-plus,body.sns-chat-page #sns-composer-ui>.sns-ui-format,body.sns-chat-page #sns-composer-ui>.sns-ui-input-wrap,body.sns-chat-page #sns-composer-ui>.sns-ui-send{grid-row:2!important;}", "body.sns-chat-page .sns-action-reply-icon{display:inline-flex;align-items:center;justify-content:center;flex:0 0 18px;width:18px;height:18px;color:currentColor;}", "body.sns-chat-page .sns-action-reply-icon svg{display:block;width:18px;height:18px;fill:none;stroke:currentColor;stroke-width:2.25;stroke-linecap:round;stroke-linejoin:round;}", "body.sns-chat-page .sns-direct-reply{position:absolute!important;top:50%!important;left:0!important;display:flex!important;align-items:center!important;justify-content:center!important;width:26px!important;height:26px!important;margin:0!important;padding:0!important;transform:translateY(-50%)!important;border:0!important;border-radius:50%!important;background:transparent!important;color:rgba(255,255,255,.92)!important;filter:drop-shadow(0 1px 2px rgba(20,20,20,.42))!important;box-shadow:none!important;cursor:pointer!important;transition:color .15s ease,transform .15s ease,opacity .15s ease!important;}", "body.sns-chat-page .sns-direct-reply:hover{color:#fff!important;filter:drop-shadow(0 1px 3px rgba(20,20,20,.58))!important;transform:translateY(-50%) scale(1.10)!important;}", "body.sns-chat-page .sns-direct-reply:active{transform:translateY(-50%) scale(.94)!important;}", "body.sns-chat-page.sns-chat-readonly .sns-action-reply{display:none!important;}", "body.sns-chat-page #sns-pending-stack .sns-reply-preview{pointer-events:none;}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-has-actions{margin-bottom:24px!important;}", "@media(max-width:650px){body.sns-chat-page #sns-reply-compose{margin-bottom:6px;padding:6px 7px;}body.sns-chat-page #pun-viewtopic .sns-message.sns-has-actions{margin-bottom:22px!important;}}" ].join("");
        document.head.appendChild(replyStyle);
      }
    })();
    (function($) {
      "use strict";
      var SNS_FORUM_ID = 4;
      function getForumId() {
        if (typeof window.ForumID !== "undefined") {
          return Number(window.ForumID);
        }
        var href = $('#pun-crumbs1 a[href*="viewforum.php?id="], ' + '#pun-crumbs2 a[href*="viewforum.php?id="]').last().attr("href") || "";
        var match = href.match(/[?&]id=(\d+)/);
        return match ? Number(match[1]) : 0;
      }
      if (!/viewtopic\.php/i.test(location.pathname) || getForumId() !== SNS_FORUM_ID) {
        return;
      }
      var snsPageMatch = String(location.search || "").match(/(?:\?|&)p=(\d+)/);
      if (snsPageMatch && parseInt(snsPageMatch[1], 10) > 1) {
        var canonicalUrl = new URL(location.href);
        canonicalUrl.searchParams.delete("p");
        canonicalUrl.hash = "";
        location.replace(canonicalUrl.toString());
        return;
      }
      var $view = $("#pun-viewtopic");
      var $topic = $view.find(".topic").first();
      if (!$view.length || !$topic.length) {
        return;
      }
      $("body").addClass("sns-chat-page");
      window.__SNS_REAL_TECH_POST_ID__ = window.__SNS_REAL_TECH_POST_ID__ || String($topic.find(".post").first().attr("id") || "");
      window.__SNS_RAW_POST_OBSERVER__ = window.__SNS_RAW_POST_OBSERVER__ || {
        disabled: true
      };
      var snsPhoneDevice = /iPhone|iPod|Android.+Mobile|Windows Phone/i.test(navigator.userAgent || "") || navigator.maxTouchPoints > 0 && Math.min(screen.width || 9999, screen.height || 9999) <= 600;
      if (snsPhoneDevice) {
        $("body").addClass("sns-phone-device");
      }
      function cleanText(text) {
        return String(text || "").replace(/\s+/g, " ").trim();
      }
      function escapeAttribute(value) {
        return String(value || "").replace(/&/g, "&amp;").replace(/"/g, "&quot;").replace(/</g, "&lt;").replace(/>/g, "&gt;");
      }
      function getCurrentUser() {
        if (typeof window.UserLogin !== "undefined" && window.UserLogin) {
          return cleanText(window.UserLogin);
        }
        var text = cleanText($("#pun-status .item1, #pun-status .status_user").first().text());
        var match = text.match(/(?:\u043F\u0440\u0438\u0432\u0435\u0442|hello)[,\s]+([^!,]+)/i);
        return match ? cleanText(match[1]) : "";
      }
      function getPostAuthor($post) {
        var name = cleanText($post.find(".pa-author a").first().text());
        if (!name) {
          name = cleanText($post.find(".pa-author").first().text());
        }
        name = cleanText(String(name || "").replace(/^\u0410\u0432\u0442\u043E\u0440:\s*/i, ""));
        return name;
      }
      function getPostAvatar($post) {
        var selectors = [ ".pa-avatar img", "img.avatardemo", ".post-author .pa-avatar img", ".post-author img.avatar" ];
        for (var i = 0; i < selectors.length; i++) {
          var src = $post.find(selectors[i]).first().attr("src");
          if (src) {
            return src;
          }
        }
        var fallback = "";
        $post.find(".post-author img[src]").each(function() {
          var $img = $(this);
          var src = String($img.attr("src") || "");
          var cls = String($img.attr("class") || "");
          if (!src || /flag|icon|online|offline|rank|smil|emoji/i.test(src + " " + cls)) {
            return;
          }
          fallback = src;
          return false;
        });
        return fallback;
      }
      function getPostTime($post) {
        var raw = cleanText($post.find("h3").first().text()).replace(/\u041F\u043E\u0434\u0435\u043B\u0438\u0442\u044C\u0441\u044F/gi, "");
        var match = raw.match(/(?:\u0421\u0435\u0433\u043E\u0434\u043D\u044F|\u0412\u0447\u0435\u0440\u0430)\s+\d{1,2}:\d{2}(?::\d{2})?|(?:\d{1,2}[.\-/]\d{1,2}[.\-/]\d{2,4})\s+\d{1,2}:\d{2}(?::\d{2})?/i);
        if (match) {
          return match[0];
        }
        return raw.replace(/^\s*\d+\s*/, "").trim();
      }
      function getTopicTitle() {
        var title = "";
        var selectors = [ "#pun-viewtopic .main-head h1 span", "#pun-viewtopic .main-head h1", "#pun-viewtopic h1 span", "#pun-viewtopic h1" ];
        for (var i = 0; i < selectors.length; i++) {
          title = cleanText($(selectors[i]).first().text());
          if (title) {
            break;
          }
        }
        if (!title) {
          var crumbs = cleanText($("#pun-crumbs1 .crumbs").first().text());
          if (crumbs) {
            var bits = crumbs.split("\xbb");
            title = cleanText(bits[bits.length - 1]);
          }
        }
        return title || "SNS CHAT";
      }
      var $firstPost = $topic.find(".post").first();
      var owner = getPostAuthor($firstPost) || "\u0432\u043b\u0430\u0434\u0435\u043b\u0435\u0446";
      var ownerAvatar = getPostAvatar($firstPost);
      function createHeader() {
        if ($("#sns-chat-header").length) {
          return;
        }
        var avatarHtml;
        if (ownerAvatar) {
          avatarHtml = "<img " + 'class="sns-head-avatar" ' + 'src="' + escapeAttribute(ownerAvatar) + '" alt="">';
        } else {
          avatarHtml = "<span " + 'class="sns-head-avatar sns-head-avatar-empty">' + (owner.charAt(0) || "S").toUpperCase() + "</span>";
        }
        var $header = $('<div id="sns-chat-header">' + '<div class="sns-head-left">' + '<span class="sns-head-avatar-box">' + avatarHtml + "</span>" + '<div class="sns-head-text">' + '<div class="sns-head-title"></div>' + '<div class="sns-head-subtitle"></div>' + "</div>" + "</div>" + '<div class="sns-head-badge">' + "SNS" + "</div>" + "</div>");
        $header.find(".sns-head-title").text(getTopicTitle());
        $header.find(".sns-head-subtitle").text("\u0441\u043e\u0437\u0434\u0430\u0442\u0435\u043b\u044c: " + owner);
        $topic.before($header);
      }
      function createShell() {
        if ($("#sns-chat-shell").length) {
          return;
        }
        var $header = $("#sns-chat-header");
        if (!$header.length || !$topic.length) {
          return;
        }
        $header.add($topic).wrapAll('<div id="sns-chat-shell"></div>');
      }
      function findActionLink($post, type) {
        var regexp = type === "edit" ? /\u0440\u0435\u0434\u0430\u043A\u0442|edit/i : /\u0443\u0434\u0430\u043B|delete/i;
        var $result = $();
        $post.find(".post-links a").each(function() {
          var $link = $(this);
          var text = cleanText($link.text());
          var href = $link.attr("href") || "";
          var match = regexp.test(text) || type === "edit" && /edit\.php|action=edit/i.test(href) || type === "delete" && /delete/i.test(href);
          if (match) {
            $result = $link;
            return false;
          }
        });
        return $result;
      }
      var snsViewer = {
        urls: [],
        index: 0
      };
      function snsImageUrl($img) {
        var $link = $img.closest("a");
        var href = $link.attr("href") || "";
        if (/^https?:\/\//i.test(href)) {
          return href;
        }
        return $img.attr("data-src") || $img.attr("src") || "";
      }
      function ensureSnsViewer() {
        var $viewer = $("#sns-photo-viewer");
        if ($viewer.length) {
          return $viewer;
        }
        $viewer = $('<div id="sns-photo-viewer" aria-hidden="true">' + '<button type="button" class="sns-viewer-close">&times;</button>' + '<button type="button" class="sns-viewer-prev">\u2039</button>' + '<div class="sns-viewer-stage">' + '<img class="sns-viewer-image" alt="">' + "</div>" + '<button type="button" class="sns-viewer-next">\u203a</button>' + '<div class="sns-viewer-count"></div>' + "</div>");
        $("body").append($viewer);
        return $viewer;
      }
      function renderSnsViewer() {
        var $viewer = ensureSnsViewer();
        var urls = snsViewer.urls;
        if (!urls.length) {
          return;
        }
        $viewer.find(".sns-viewer-image").attr("src", urls[snsViewer.index]);
        $viewer.find(".sns-viewer-count").text(urls.length > 1 ? snsViewer.index + 1 + " / " + urls.length : "");
        $viewer.find(".sns-viewer-prev, .sns-viewer-next").toggle(urls.length > 1);
      }
      function openSnsViewer($img) {
        var $content = $img.closest(".post-content");
        var urls = [];
        var current = snsImageUrl($img);
        $content.find("img").not('.smalimg, .sns-inline-smilie, [alt="smalimg"], [title="smalimg"]').each(function() {
          var url = snsImageUrl($(this));
          if (url && $.inArray(url, urls) === -1) {
            urls.push(url);
          }
        });
        if (!urls.length && current) {
          urls.push(current);
        }
        snsViewer.urls = urls;
        var found = $.inArray(current, urls);
        snsViewer.index = found >= 0 ? found : 0;
        var $viewer = ensureSnsViewer();
        renderSnsViewer();
        $viewer.addClass("is-open").attr("aria-hidden", "false");
        $("body").addClass("sns-viewer-open");
      }
      function closeSnsViewer() {
        $("#sns-photo-viewer").removeClass("is-open").attr("aria-hidden", "true").find(".sns-viewer-image").attr("src", "");
        $("body").removeClass("sns-viewer-open");
      }
      function stepSnsViewer(direction) {
        var count = snsViewer.urls.length;
        if (count < 2) {
          return;
        }
        snsViewer.index = (snsViewer.index + direction + count) % count;
        renderSnsViewer();
      }
      $(document).off(".snsPhotoViewer").on("click.snsPhotoViewer", '.sns-message .post-content img:not(.smalimg):not(.sns-inline-smilie):not([alt="smalimg"]):not([title="smalimg"])', function(event) {
        event.preventDefault();
        event.stopPropagation();
        openSnsViewer($(this));
      }).on("click.snsPhotoViewer", ".sns-viewer-close", function() {
        closeSnsViewer();
      }).on("click.snsPhotoViewer", ".sns-viewer-prev", function(event) {
        event.stopPropagation();
        stepSnsViewer(-1);
      }).on("click.snsPhotoViewer", ".sns-viewer-next", function(event) {
        event.stopPropagation();
        stepSnsViewer(1);
      }).on("click.snsPhotoViewer", "#sns-photo-viewer", function(event) {
        if (event.target === this) {
          closeSnsViewer();
        }
      }).on("keydown.snsPhotoViewer", function(event) {
        if (!$("#sns-photo-viewer").hasClass("is-open")) {
          return;
        }
        if (event.key === "Escape") {
          closeSnsViewer();
        } else if (event.key === "ArrowLeft") {
          stepSnsViewer(-1);
        } else if (event.key === "ArrowRight") {
          stepSnsViewer(1);
        }
      });
      var SNS_VOICE_PREFIX = "SNSVOICE:";
      function encodeVoiceText(value) {
        value = String(value || "");
        try {
          return btoa(unescape(encodeURIComponent(value)));
        } catch (error) {
          return "";
        }
      }
      function decodeVoiceText(encoded) {
        encoded = String(encoded || "");
        if (!encoded) {
          return null;
        }
        try {
          return decodeURIComponent(escape(atob(encoded)));
        } catch (error) {
          return null;
        }
      }
      function voiceMarkerFromText(value) {
        var encoded = encodeVoiceText(String(value || "").trim());
        return encoded ? SNS_VOICE_PREFIX + encoded : "";
      }
      function parseVoiceMarker(value) {
        var raw = String(value || "");
        var match = raw.match(/SNSVOICE:([A-Za-z0-9+\/=]+)/);
        if (!match) {
          return null;
        }
        var decoded = decodeVoiceText(match[1]);
        if (decoded === null) {
          return null;
        }
        return {
          encoded: match[1],
          text: decoded
        };
      }
      function voiceDurationLabel(value) {
        var words = String(value || "").trim().split(/\s+/).filter(Boolean).length;
        var seconds = Math.max(2, Math.ceil(words / 2.25));
        seconds = Math.min(599, seconds);
        var minutes = Math.floor(seconds / 60);
        var rest = seconds % 60;
        return minutes + ":" + (rest < 10 ? "0" : "") + rest;
      }
      function voiceWaveMarkup(value) {
        var source = String(value || "voice");
        var seed = 17;
        for (var i = 0; i < source.length; i++) {
          seed = (seed * 31 + source.charCodeAt(i)) % 9973;
        }
        var html = "";
        for (var index = 0; index < 44; index++) {
          seed = (seed * 37 + index * 19 + 11) % 9973;
          var height = 4 + seed % 17;
          html += '<i style="--sns-vh:' + height + 'px"></i>';
        }
        return html;
      }
      function createVoiceCard(value, expanded) {
        var textValue = String(value || "");
        var $card = $('<div class="sns-voice-card">' + '<button type="button" class="sns-voice-toggle" aria-expanded="false">' + '<span class="sns-voice-play">' + '<svg class="sns-voice-play-icon" viewBox="0 0 24 24" aria-hidden="true" focusable="false">' + '<path d="M8 5v14l11-7z"></path>' + "</svg>" + '<svg class="sns-voice-pause-icon" viewBox="0 0 24 24" aria-hidden="true" focusable="false">' + '<path d="M7 5h4v14H7zM13 5h4v14h-4z"></path>' + "</svg>" + "</span>" + '<span class="sns-voice-wave"></span>' + '<span class="sns-voice-time"></span>' + "</button>" + '<div class="sns-voice-transcript"></div>' + "</div>");
        $card.find(".sns-voice-wave").html(voiceWaveMarkup(textValue));
        $card.find(".sns-voice-time").text(voiceDurationLabel(textValue));
        $card.find(".sns-voice-transcript").text(textValue);
        if (expanded) {
          $card.addClass("is-open").find(".sns-voice-toggle").attr("aria-expanded", "true");
        }
        return $card;
      }
      function renderVoicePost($post, $postContent, payload) {
        if (!$post || !$post.length || !$postContent || !$postContent.length || !payload) {
          return false;
        }
        var wasOpen = $post.hasClass("sns-voice-expanded");
        $post.addClass("sns-voice-message").removeClass("sns-has-media sns-media-only sns-media-caption sns-image-only").attr("data-sns-voice", payload.encoded).attr("data-sns-images", "0");
        $postContent.empty().append(createVoiceCard(payload.text, wasOpen));
        return true;
      }
      window.SNSVoiceEncodeMarker = voiceMarkerFromText;
      window.SNSVoiceDecodeMarker = function(value) {
        var parsed = parseVoiceMarker(value);
        return parsed ? parsed.text : null;
      };
      function cloudinaryAudioConfig() {
        var raw = window.SNS_AUDIO_UPLOAD_CONFIG && typeof window.SNS_AUDIO_UPLOAD_CONFIG === "object" ? window.SNS_AUDIO_UPLOAD_CONFIG : {};
        var maxMb = Number(raw.maxMb);
        if (!isFinite(maxMb) || maxMb <= 0) {
          maxMb = 15;
        }
        maxMb = Math.max(1, Math.min(100, maxMb));
        return {
          cloudName: String(raw.cloudName || "").trim(),
          uploadPreset: String(raw.uploadPreset || "").trim(),
          maxMb: maxMb
        };
      }
      function cloudinaryAudioReady() {
        var cfg = cloudinaryAudioConfig();
        return !!(cfg.cloudName && cfg.uploadPreset);
      }
      function audioUploadFileAllowed(file) {
        if (!file) {
          return false;
        }
        var name = String(file.name || "");
        var mime = String(file.type || "");
        return /^audio\//i.test(mime) || /\.(?:mp3|ogg|oga|wav|m4a|aac|flac)$/i.test(name);
      }
      function audioTitleFromFileName(name) {
        return String(name || "").replace(/\.(?:mp3|ogg|oga|wav|m4a|aac|flac)$/i, "").replace(/[_-]+/g, " ").trim();
      }
      var SNS_AUDIO_PREFIX = "SNSAUDIO:";
      function audioSafeUrl(value) {
        var raw = String(value || "").trim();
        if (!/^https?:\/\//i.test(raw)) {
          return "";
        }
        return raw;
      }
      function audioFallbackTitle(url) {
        var result = "\u0410\u0443\u0434\u0438\u043e";
        try {
          var parsed = new URL(url, location.href);
          var fileName = decodeURIComponent(parsed.pathname.split("/").pop() || "").replace(/\.(?:mp3|ogg|oga|wav|m4a|aac|flac)$/i, "").replace(/[_-]+/g, " ").trim();
          if (fileName) {
            result = fileName;
          }
        } catch (error) {}
        return result;
      }
      function normalizeAudioData(data) {
        data = data && typeof data === "object" ? data : {};
        var url = audioSafeUrl(data.url);
        if (!url) {
          return null;
        }
        return {
          url: url,
          title: String(data.title || "").trim() || audioFallbackTitle(url),
          artist: String(data.artist || "").trim(),
          cover: audioSafeUrl(data.cover)
        };
      }
      function audioMarkerFromData(data) {
        var normalized = normalizeAudioData(data);
        if (!normalized) {
          return "";
        }
        var encoded = encodeVoiceText(JSON.stringify(normalized));
        return encoded ? SNS_AUDIO_PREFIX + encoded : "";
      }
      function parseAudioMarker(value) {
        var raw = String(value || "");
        var match = raw.match(/SNSAUDIO:([A-Za-z0-9+\/=]+)/);
        if (!match) {
          return null;
        }
        var decoded = decodeVoiceText(match[1]);
        if (decoded === null) {
          return null;
        }
        try {
          var data = normalizeAudioData(JSON.parse(decoded));
          if (!data) {
            return null;
          }
          return {
            encoded: match[1],
            data: data
          };
        } catch (error) {
          return null;
        }
      }
      function formatAudioClock(value) {
        var seconds = Number(value);
        if (!isFinite(seconds) || seconds < 0) {
          seconds = 0;
        }
        seconds = Math.floor(seconds);
        var minutes = Math.floor(seconds / 60);
        var rest = seconds % 60;
        return minutes + ":" + (rest < 10 ? "0" : "") + rest;
      }
      function stopAudioProgressTicker(node) {
        if (!node) {
          return;
        }
        if (node.__snsAudioRaf) {
          cancelAnimationFrame(node.__snsAudioRaf);
          node.__snsAudioRaf = 0;
        }
      }
      function startAudioProgressTicker(node) {
        stopAudioProgressTicker(node);
        if (node && !document.hidden) updateAudioCard($(node));
      }
      function bindAudioNativeEvents($audio) {
        if (!$audio || !$audio.length) {
          return;
        }
        var node = $audio.get(0);
        if (!node || node.__snsAudioBound) {
          return;
        }
        node.__snsAudioBound = true;
        [ "loadedmetadata", "durationchange", "timeupdate", "seeking", "seeked", "ratechange" ].forEach(function(eventName) {
          node.addEventListener(eventName, function() {
            updateAudioCard($(node));
          });
        });
        node.addEventListener("play", function() {
          updateAudioCard($(node));
          startAudioProgressTicker(node);
        });
        node.addEventListener("pause", function() {
          stopAudioProgressTicker(node);
          updateAudioCard($(node));
        });
        node.addEventListener("ended", function() {
          stopAudioProgressTicker(node);
          updateAudioCard($(node));
        });
        node.addEventListener("error", function() {
          stopAudioProgressTicker(node);
          $(node).closest(".sns-audio-card").addClass("has-error");
          updateAudioCard($(node));
        });
      }
      function createAudioCard(data) {
        var normalized = normalizeAudioData(data);
        if (!normalized) {
          return $("<div></div>");
        }
        var $card = $('<div class="sns-audio-card">' + '<div class="sns-audio-cover"></div>' + '<div class="sns-audio-main">' + '<span class="sns-audio-title"></span>' + '<span class="sns-audio-artist"></span>' + '<div class="sns-audio-controls">' + '<button type="button" class="sns-audio-play" title="\u0412\u043e\u0441\u043f\u0440\u043e\u0438\u0437\u0432\u0435\u0441\u0442\u0438">' + '<svg class="sns-audio-play-icon" viewBox="0 0 24 24" aria-hidden="true"><path d="M8 5v14l11-7z"></path></svg>' + '<svg class="sns-audio-pause-icon" viewBox="0 0 24 24" aria-hidden="true"><path d="M7 5h4v14H7zM13 5h4v14h-4z"></path></svg>' + "</button>" + '<div class="sns-audio-track" role="slider" aria-label="\u041f\u043e\u0437\u0438\u0446\u0438\u044f \u0430\u0443\u0434\u0438\u043e" tabindex="0">' + '<span class="sns-audio-progress"></span>' + "</div>" + '<span class="sns-audio-time">0:00</span>' + "</div>" + "</div>" + '<div class="sns-audio-error">\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u0430\u0443\u0434\u0438\u043e</div>' + '<audio class="sns-audio-native" preload="none"></audio>' + "</div>");
        $card.find(".sns-audio-title").text(normalized.title);
        $card.find(".sns-audio-artist").text(normalized.artist || "\u0430\u0443\u0434\u0438\u043e");
        var $cover = $card.find(".sns-audio-cover");
        if (normalized.cover) {
          $('<img alt="">').attr("src", normalized.cover).appendTo($cover);
        } else {
          $cover.html('<svg viewBox="0 0 24 24" aria-hidden="true">' + '<path d="M9 18V5l10-2v13"></path>' + '<circle cx="6" cy="18" r="3"></circle>' + '<circle cx="16" cy="16" r="3"></circle>' + "</svg>");
        }
        var $nativeAudio = $card.find(".sns-audio-native").attr("src", normalized.url);
        bindAudioNativeEvents($nativeAudio);
        $card.attr("data-sns-audio-url", normalized.url);
        return $card;
      }
      function renderAudioPost($post, $postContent, payload) {
        if (!$post || !$post.length || !$postContent || !$postContent.length || !payload || !payload.data) {
          return false;
        }
        $post.addClass("sns-audio-message").removeClass("sns-voice-message sns-voice-expanded sns-has-media sns-media-only sns-media-caption sns-image-only").removeAttr("data-sns-voice").attr("data-sns-audio", payload.encoded).attr("data-sns-images", "0");
        $postContent.empty().append(createAudioCard(payload.data));
        return true;
      }
      window.SNSAudioEncodeMarker = audioMarkerFromData;
      window.SNSAudioDecodeMarker = function(value) {
        var parsed = parseAudioMarker(value);
        return parsed ? parsed.data : null;
      };
      window.SNSAudioEditText = function(data) {
        var normalized = normalizeAudioData(data);
        if (!normalized) {
          return "";
        }
        return [ "\u0421\u0441\u044b\u043b\u043a\u0430: " + normalized.url, "\u041d\u0430\u0437\u0432\u0430\u043d\u0438\u0435: " + normalized.title, "\u0418\u0441\u043f\u043e\u043b\u043d\u0438\u0442\u0435\u043b\u044c: " + normalized.artist, "\u041e\u0431\u043b\u043e\u0436\u043a\u0430: " + normalized.cover ].join("\n");
      };
      window.SNSAudioParseEditText = function(value) {
        var lines = String(value || "").split(/\r?\n/);
        var data = {
          url: "",
          title: "",
          artist: "",
          cover: ""
        };
        lines.forEach(function(line, index) {
          var textLine = String(line || "").trim();
          var match = textLine.match(/^([^:]+):\s*(.*)$/);
          if (match) {
            var key = match[1].toLowerCase().trim();
            var val = match[2].trim();
            if (key === "\u0441\u0441\u044b\u043b\u043a\u0430" || key === "url") {
              data.url = val;
            } else if (key === "\u043d\u0430\u0437\u0432\u0430\u043d\u0438\u0435") {
              data.title = val;
            } else if (key === "\u0438\u0441\u043f\u043e\u043b\u043d\u0438\u0442\u0435\u043b\u044c") {
              data.artist = val;
            } else if (key === "\u043e\u0431\u043b\u043e\u0436\u043a\u0430") {
              data.cover = val;
            }
          } else if (index === 0 && /^https?:\/\//i.test(textLine)) {
            data.url = textLine;
          }
        });
        return normalizeAudioData(data);
      };
      var SNS_STORY_PREFIX = "SNSSTORY:";
      var snsStoryDate = "";
      var snsStoryDateMode = 0;
      var snsStoryNextTime = "";
      var snsStoryUserTouched = false;
      function normalizeStoryMeta(data) {
        data = data && typeof data === "object" ? data : {};
        var dateSet = data.dateSet === true || data.dateSet === 1 || data.ds === true || data.ds === 1;
        var date = String(data.date !== undefined ? data.date : data.d !== undefined ? data.d : "").replace(/\s+/g, " ").trim().slice(0, 90);
        var time = String(data.time !== undefined ? data.time : data.t !== undefined ? data.t : "").replace(/\s+/g, " ").trim().slice(0, 40);
        if (!dateSet && !time) {
          return null;
        }
        return {
          dateSet: dateSet,
          date: date,
          time: time
        };
      }
      function encodeStoryMeta(data) {
        var normalized = normalizeStoryMeta(data);
        if (!normalized) {
          return "";
        }
        return encodeVoiceText(JSON.stringify({
          ds: normalized.dateSet ? 1 : 0,
          d: normalized.date,
          t: normalized.time
        }));
      }
      function decodeStoryMeta(encoded) {
        var decoded = decodeVoiceText(encoded);
        if (decoded === null) {
          return null;
        }
        try {
          return normalizeStoryMeta(JSON.parse(decoded));
        } catch (error) {
          return null;
        }
      }
      function wrapStoryMessage(meta, body) {
        var encoded = encodeStoryMeta(meta);
        if (!encoded) {
          return String(body || "");
        }
        return SNS_STORY_PREFIX + encoded + "\n" + String(body || "");
      }
      function parseStoryWrappedRaw(value) {
        var match = String(value || "").match(/^\s*SNSSTORY:([A-Za-z0-9+\/=]+)[ \t]*(?:\r?\n)?([\s\S]*)$/);
        if (!match) {
          return null;
        }
        var meta = decodeStoryMeta(match[1]);
        return meta ? {
          encoded: match[1],
          meta: meta,
          body: String(match[2] || "")
        } : null;
      }
      function parseStoryMetaFromRendered(value) {
        var match = String(value || "").match(/SNSSTORY:([A-Za-z0-9+\/=]+)/);
        if (!match) {
          return null;
        }
        var meta = decodeStoryMeta(match[1]);
        return meta ? {
          encoded: match[1],
          meta: meta
        } : null;
      }
      function storyMetaForNextMessage() {
        var time = String(snsStoryNextTime || "").replace(/\s+/g, " ").trim();
        if (snsStoryDateMode === 1) {
          return {
            dateSet: true,
            date: snsStoryDate,
            time: time
          };
        }
        if (snsStoryDateMode === 2) {
          return {
            dateSet: true,
            date: "",
            time: time
          };
        }
        if (time) {
          return {
            dateSet: false,
            date: "",
            time: time
          };
        }
        return null;
      }
      function storyMetaFromPost($post) {
        if (!$post || !$post.length) {
          return null;
        }
        var encoded = String($post.attr("data-sns-story") || "");
        return encoded ? decodeStoryMeta(encoded) : null;
      }
      function renderStoryDateSeparators() {
        var $topic = $("#sns-chat-shell>.topic").first();
        if (!$topic.length) {
          return;
        }
        $topic.children(".sns-story-date-separator").remove();
        var currentDate = "";
        var displayedDate = null;
        $topic.children(".post.sns-message").each(function() {
          var $post = $(this);
          var meta = storyMetaFromPost($post);
          if (meta && meta.dateSet) {
            currentDate = meta.date;
          }
          if (currentDate && currentDate !== displayedDate) {
            $('<div class="sns-story-date-separator" aria-hidden="true">' + "<span></span>" + "</div>").find("span").text(currentDate).end().insertBefore($post);
            displayedDate = currentDate;
          } else if (!currentDate) {
            displayedDate = "";
          }
        });
      }
      function syncStoryDraftFromHistory() {
        if (snsStoryUserTouched) {
          return;
        }
        var found = false;
        var latestDate = "";
        $("#sns-chat-shell>.topic>.post.sns-message").each(function() {
          var meta = storyMetaFromPost($(this));
          if (meta && meta.dateSet) {
            found = true;
            latestDate = meta.date;
          }
        });
        if (!found) {
          snsStoryDate = "";
          snsStoryDateMode = 0;
          return;
        }
        snsStoryDate = latestDate;
        snsStoryDateMode = latestDate ? 1 : 0;
      }
      var SNS_REPLY_PREFIX = "SNSREPLY:";
      var snsReplyDraft = null;
      function normalizeReplyMeta(data) {
        data = data && typeof data === "object" ? data : {};
        var targetId = String(data.targetId || "").replace(/\D+/g, "");
        if (!targetId) {
          return null;
        }
        return {
          targetId: targetId,
          author: String(data.author || "\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435").trim().slice(0, 80),
          preview: String(data.preview || "\u0421\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435").trim().replace(/\s+/g, " ").slice(0, 120)
        };
      }
      function encodeReplyMeta(data) {
        var normalized = normalizeReplyMeta(data);
        if (!normalized) {
          return "";
        }
        return encodeVoiceText(JSON.stringify(normalized));
      }
      function decodeReplyMeta(encoded) {
        var decoded = decodeVoiceText(encoded);
        if (decoded === null) {
          return null;
        }
        try {
          return normalizeReplyMeta(JSON.parse(decoded));
        } catch (error) {
          return null;
        }
      }
      function wrapReplyMessage(meta, body) {
        var encoded = encodeReplyMeta(meta);
        if (!encoded) {
          return String(body || "");
        }
        return SNS_REPLY_PREFIX + encoded + "\n" + String(body || "");
      }
      function parseReplyWrappedRaw(value) {
        var match = String(value || "").match(/^\s*SNSREPLY:([A-Za-z0-9+\/=]+)[ \t]*(?:\r?\n)?([\s\S]*)$/);
        if (!match) {
          return null;
        }
        var meta = decodeReplyMeta(match[1]);
        return meta ? {
          encoded: match[1],
          meta: meta,
          body: String(match[2] || "")
        } : null;
      }
      function parseReplyMetaFromRendered(value) {
        var match = String(value || "").match(/SNSREPLY:([A-Za-z0-9+\/=]+)/);
        if (!match) {
          return null;
        }
        var meta = decodeReplyMeta(match[1]);
        return meta ? {
          encoded: match[1],
          meta: meta
        } : null;
      }
      function stripReplyMarkerFromContent($content, marker) {
        var root = $content && $content.length ? $content.get(0) : null;
        if (!root || !marker) {
          return;
        }
        var walker = document.createTreeWalker(root, NodeFilter.SHOW_TEXT, null);
        var node;
        while (node = walker.nextNode()) {
          var value = String(node.nodeValue || "");
          var index = value.indexOf(marker);
          if (index !== -1) {
            node.nodeValue = value.slice(0, index) + value.slice(index + marker.length);
            break;
          }
        }
        function trimLeadingReplyBreaks(node, depth) {
          if (!node || depth > 4) {
            return;
          }
          while (node.firstChild) {
            var first = node.firstChild;
            if (first.nodeType === 3 && !String(first.nodeValue || "").trim()) {
              node.removeChild(first);
              continue;
            }
            if (first.nodeType === 1 && String(first.tagName || "").toLowerCase() === "br") {
              node.removeChild(first);
              continue;
            }
            if (first.nodeType === 1 && /^(p|div|span)$/i.test(String(first.tagName || ""))) {
              trimLeadingReplyBreaks(first, depth + 1);
              if (!String(first.textContent || "").trim() && !first.querySelector("img,iframe,video,audio")) {
                node.removeChild(first);
                continue;
              }
            }
            break;
          }
        }
        trimLeadingReplyBreaks(root, 0);
        $content.children().each(function() {
          var $child = $(this);
          if ($child.is(".sns-reply-preview")) {
            return;
          }
          var clone = $child.clone();
          clone.find("br").remove();
          if (!cleanText(clone.text()) && !$child.find("img,iframe,video,audio").length) {
            $child.remove();
            return;
          }
          return false;
        });
      }
      function replyPreviewForPost($post) {
        if ($post.hasClass("sns-voice-message")) {
          return "\u0413\u043e\u043b\u043e\u0441\u043e\u0432\u043e\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435";
        }
        if ($post.hasClass("sns-audio-message")) {
          var audioTitle = cleanText($post.find(".sns-audio-title").first().text());
          return audioTitle ? "\u266b " + audioTitle : "\u0410\u0443\u0434\u0438\u043e";
        }
        var imageCount = parseInt($post.attr("data-sns-images"), 10) || $post.find(".post-content img").length;
        if (imageCount > 0) {
          return imageCount === 1 ? "\u0424\u043e\u0442\u043e" : imageCount + " \u0444\u043e\u0442\u043e";
        }
        if ($post.find(".post-content iframe, .post-content video").length) {
          return "\u0412\u0438\u0434\u0435\u043e";
        }
        var $clone = $post.find(".post-content").first().clone();
        $clone.find(".sns-reply-preview, " + ".sns-media-grid, " + ".sns-source-media-hidden, " + ".sns-reaction-chips, " + ".sns-reaction-chip, " + ".sns-reaction-control, " + ".sns-reaction-picker-custom, " + ".sns-reaction-error, " + "audio").remove();
        var preview = cleanText($clone.text()) || "\u0421\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435";
        return preview.length > 88 ? preview.slice(0, 85) + "\u2026" : preview;
      }
      function replySnapshotForPost($post) {
        var targetId = String($post.attr("id") || "").replace(/^p/i, "").replace(/\D+/g, "");
        return targetId ? normalizeReplyMeta({
          targetId: targetId,
          author: getPostAuthor($post) || "\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435",
          preview: replyPreviewForPost($post)
        }) : null;
      }
      function createReplyPreview(meta) {
        var normalized = normalizeReplyMeta(meta);
        if (!normalized) {
          return $();
        }
        return $('<button type="button" class="sns-reply-preview"></button>').attr("data-sns-reply-target", normalized.targetId).append($('<span class="sns-reply-preview-author"></span>').text(normalized.author), $('<span class="sns-reply-preview-text"></span>').text(normalized.preview));
      }
      function renderReplyPreview($post, $content, meta) {
        $content.children(".sns-reply-preview").remove();
        var normalized = normalizeReplyMeta(meta);
        if (!normalized) {
          $post.removeClass("sns-has-reply").removeAttr("data-sns-reply data-sns-reply-target");
          return;
        }
        $post.addClass("sns-has-reply").attr("data-sns-reply-target", normalized.targetId);
        $content.prepend(createReplyPreview(normalized));
      }
      function updateReplyComposeBar() {
        var $bar = $("#sns-reply-compose");
        if (!$bar.length) {
          return;
        }
        $bar.toggleClass("is-open", !!snsReplyDraft).attr("aria-hidden", snsReplyDraft ? "false" : "true");
        if (!snsReplyDraft) {
          return;
        }
        $bar.find(".sns-reply-compose-author").text(snsReplyDraft.author);
        $bar.find(".sns-reply-compose-text").text(snsReplyDraft.preview);
      }
      function clearReplyDraft() {
        snsReplyDraft = null;
        updateReplyComposeBar();
      }
      function setReplyDraftFromPost($post) {
        var snapshot = replySnapshotForPost($post);
        if (!snapshot) {
          return;
        }
        snsReplyDraft = snapshot;
        updateReplyComposeBar();
        $("#sns-ui-input").focus();
      }
      function normalizeNativeAjaxPosts() {
        if (!$topic || !$topic.length) {
          return 0;
        }
        var normalized = 0;
        $topic.find(".post.new-ajax").each(function() {
          var $post = $(this);
          var rawId = String($post.attr("id") || "");
          if (rawId) {
            var $same = $topic.find(".post#" + rawId.replace(/[^A-Za-z0-9_-]/g, ""));
            if ($same.length > 1) {
              var $existingEnhanced = $same.filter(".sns-message").not($post).first();
              if ($existingEnhanced.length) {
                $post.remove();
                return;
              }
              $same.not($post).each(function() {
                var $other = $(this);
                if (!$other.hasClass("sns-message")) {
                  $other.remove();
                }
              });
            }
          }
          $post.removeClass("new-ajax").removeAttr("style");
          var $authorLink = $post.find(".pa-author a").first();
          if ($authorLink.length) {
            var authorText = cleanText($authorLink.text()).replace(/^\u0410\u0432\u0442\u043E\u0440:\s*/i, "");
            if (authorText) {
              $authorLink.text(authorText);
            }
          }
          normalized++;
        });
        return normalized;
      }
      function canonicalizeRusffStickerImages($scope) {
        if (!$scope || !$scope.length) {
          return $();
        }
        var $stickers = $scope.find("img.smalimg, " + "img.sns-inline-smilie, " + 'img[alt="smalimg"], ' + 'img[title="smalimg"], ' + 'img[data-alt="smalimg"], ' + 'img[data-title="smalimg"]');
        $stickers.each(function() {
          var $img = $(this);
          $img.addClass("sns-inline-smilie");
          $img.removeClass("sns-gallery-image sns-photo sns-media-image").removeAttr("data-sns-photo data-sns-gallery-index");
        });
        return $stickers;
      }
      function normalizeStickerLayout($post, $postContent) {
        if (!$post || !$post.length || !$postContent || !$postContent.length) {
          return;
        }
        var $stickers = canonicalizeRusffStickerImages($postContent);
        if (!$stickers.length) {
          $post.removeClass("sns-has-sticker sns-sticker-with-text sns-sticker-only").removeAttr("data-sns-stickers");
          return;
        }
        var $textClone = $postContent.clone();
        $textClone.find("img.smalimg, img.sns-inline-smilie, " + ".sns-reaction-chips, .sns-reply-preview, " + ".sns-media-grid, .sns-source-media-hidden").remove();
        $textClone.find("br").remove();
        var hasText = !!cleanText($textClone.text());
        $post.addClass("sns-has-sticker").toggleClass("sns-sticker-with-text", hasText).toggleClass("sns-sticker-only", !hasText).attr("data-sns-stickers", $stickers.length);
        $stickers.each(function() {
          var $img = $(this);
          if (hasText) {
            while ($img.prev().is("br")) {
              $img.prev().remove();
            }
            while ($img.next().is("br")) {
              $img.next().remove();
            }
            var $paragraph = $img.closest("p");
            if ($paragraph.length) {
              var $paragraphClone = $paragraph.clone();
              $paragraphClone.find("img.smalimg, img.sns-inline-smilie, br").remove();
              var paragraphHasText = !!cleanText($paragraphClone.text());
              if (!paragraphHasText) {
                var paragraphTextFilter = function() {
                  var $clone = $(this).clone();
                  $clone.find("img.smalimg, img.sns-inline-smilie, " + 'img[alt="smalimg"], img[title="smalimg"], br').remove();
                  return !!cleanText($clone.text());
                };
                var $previousTextParagraph = $paragraph.prevAll("p").filter(paragraphTextFilter).first();
                var $nextTextParagraph = $paragraph.nextAll("p").filter(paragraphTextFilter).first();
                if ($previousTextParagraph.length) {
                  $img.detach();
                  $previousTextParagraph.append(document.createTextNode(" "));
                  $previousTextParagraph.append($img);
                } else if ($nextTextParagraph.length) {
                  $img.detach();
                  $nextTextParagraph.prepend(document.createTextNode(" "));
                  $nextTextParagraph.prepend($img);
                }
                if ($previousTextParagraph.length || $nextTextParagraph.length) {
                  var $leftover = $paragraph.clone();
                  $leftover.find("img.smalimg, img.sns-inline-smilie, " + 'img[alt="smalimg"], img[title="smalimg"], br').remove();
                  if (!cleanText($leftover.text())) {
                    $paragraph.remove();
                  }
                }
              }
            }
          }
        });
      }
      function enhancePosts(force) {
        normalizeNativeAjaxPosts();
        var $posts = $topic.children(".post");
        if (!$posts.length) {
          return;
        }
        var technicalPostId = String(window.__SNS_REAL_TECH_POST_ID__ || "");
        var me = getCurrentUser().toLowerCase();
        var visibleCount = 0;
        $posts.each(function() {
          var $post = $(this);
          var currentPostId = String($post.attr("id") || "");
          if (technicalPostId && currentPostId === technicalPostId) {
            $post.addClass("sns-tech-post").attr("aria-hidden", "true");
            return;
          }
          if ($post.hasClass("sns-tech-post")) {
            $post.removeClass("sns-tech-post").removeAttr("aria-hidden");
          }
          var contentNode = this.querySelector(".post-content");
          var forced = force === true || force && (force === this || force.jquery && force.get(0) === this);
          if (!forced && contentNode && this.__snsEnhancedContent === contentNode && $post.hasClass("sns-message")) {
            visibleCount++;
            return;
          }
          var author = getPostAuthor($post);
          if (!author) {
            return;
          }
          var snsRawContent = cleanText($post.find(".post-content").first().text());
          if (/SNSCFG:[A-Za-z0-9+\/=]+/.test(snsRawContent)) {
            $post.addClass("sns-config-post").attr("aria-hidden", "true");
            return;
          }
          var isOwn = !!me && author.toLowerCase() === me;
          var time = getPostTime($post);
          var avatar = getPostAvatar($post);
          var $container = $post.children(".container").first();
          if (!$container.length) {
            $container = $post;
          }
          var $body = $container.find(".post-body").first();
          if (!$body.length) {
            return;
          }
          visibleCount++;
          $post.addClass("sns-message").removeClass("sns-raw-awaiting-enhance").toggleClass("sns-own", isOwn).toggleClass("sns-other", !isOwn).css({
            display: "",
            visibility: "",
            opacity: "",
            pointerEvents: ""
          });
          var $postContent = $body.find(".post-content").first();
          if ($postContent.length) {
            var storedStoryEncoded = String($post.attr("data-sns-story") || "");
            var storyPayload = storedStoryEncoded ? {
              encoded: storedStoryEncoded,
              meta: decodeStoryMeta(storedStoryEncoded)
            } : parseStoryMetaFromRendered(cleanText($postContent.text()));
            if (storyPayload && storyPayload.meta) {
              $post.attr("data-sns-story", storyPayload.encoded);
              if (!storedStoryEncoded) {
                stripReplyMarkerFromContent($postContent, SNS_STORY_PREFIX + storyPayload.encoded);
              }
            }
            var initialRenderedText = cleanText($postContent.text());
            var storedReplyEncoded = String($post.attr("data-sns-reply") || "");
            var replyPayload = storedReplyEncoded ? {
              encoded: storedReplyEncoded,
              meta: decodeReplyMeta(storedReplyEncoded)
            } : parseReplyMetaFromRendered(initialRenderedText);
            if (replyPayload && replyPayload.meta) {
              $post.attr("data-sns-reply", replyPayload.encoded);
              if (!storedReplyEncoded) {
                stripReplyMarkerFromContent($postContent, SNS_REPLY_PREFIX + replyPayload.encoded);
              }
            }
            var rawSpecialContent = cleanText($postContent.text());
            var storedAudioEncoded = String($post.attr("data-sns-audio") || "");
            var audioPayload = storedAudioEncoded ? {
              encoded: storedAudioEncoded,
              data: function() {
                var decoded = decodeVoiceText(storedAudioEncoded);
                if (decoded === null) {
                  return null;
                }
                try {
                  return normalizeAudioData(JSON.parse(decoded));
                } catch (error) {
                  return null;
                }
              }()
            } : parseAudioMarker(rawSpecialContent);
            var storedVoiceEncoded = String($post.attr("data-sns-voice") || "");
            var voicePayload = storedVoiceEncoded ? {
              encoded: storedVoiceEncoded,
              text: decodeVoiceText(storedVoiceEncoded)
            } : parseVoiceMarker(rawSpecialContent);
            if (audioPayload && audioPayload.data) {
              renderAudioPost($post, $postContent, audioPayload);
            } else if (voicePayload && voicePayload.text !== null) {
              $post.removeClass("sns-audio-message").removeAttr("data-sns-audio");
              renderVoicePost($post, $postContent, voicePayload);
            } else {
              $post.removeClass("sns-voice-message sns-voice-expanded sns-audio-message").removeAttr("data-sns-voice data-sns-audio");
              normalizeStickerLayout($post, $postContent);
              var $existingGrid = $postContent.children(".sns-media-grid").first();
              var $sourceImages = $existingGrid.length ? $existingGrid.find("img") : $postContent.find("img").not('.sns-media-grid img, img.smalimg, img.sns-inline-smilie, img[alt="smalimg"], img[title="smalimg"]');
              var imageCount = $sourceImages.length;
              var $contentClone = $postContent.clone();
              $contentClone.find(".sns-media-grid, .sns-source-media-hidden, .sns-reply-preview").remove();
              $contentClone.find("img, br").remove();
              var remainingText = cleanText($contentClone.text());
              $post.toggleClass("sns-has-media", imageCount > 0).toggleClass("sns-media-only", imageCount > 0 && !remainingText).toggleClass("sns-media-caption", imageCount > 0 && !!remainingText).toggleClass("sns-image-only", imageCount === 1 && !remainingText).attr("data-sns-images", imageCount);
              if (imageCount > 1 && !$existingGrid.length) {
                var gridClass = "sns-media-count-" + (imageCount <= 9 ? imageCount : "many");
                var $grid = $('<div class="sns-media-grid ' + gridClass + '"></div>');
                var originalImages = $sourceImages.toArray();
                $.each(originalImages, function(_, imageNode) {
                  var $img = $(imageNode);
                  var src = $img.attr("src") || $img.attr("data-src") || "";
                  if (!src) {
                    return;
                  }
                  var $cloneImg = $img.clone(false).removeAttr("id").removeAttr("style");
                  if (!$cloneImg.attr("src") && $cloneImg.attr("data-src")) {
                    $cloneImg.attr("src", $cloneImg.attr("data-src"));
                  }
                  var $link = $img.closest("a");
                  var href = $link.attr("href") || "";
                  var $mediaNode;
                  if (href) {
                    $mediaNode = $('<a class="sns-media-link"></a>').attr("href", href).append($cloneImg);
                  } else {
                    $mediaNode = $cloneImg;
                  }
                  $('<div class="sns-media-item"></div>').append($mediaNode).appendTo($grid);
                  var $originalLink = $img.closest("a");
                  if ($originalLink.length && $originalLink.find("img").length === 1) {
                    var $linkTextClone = $originalLink.clone();
                    $linkTextClone.find("img").remove();
                    if (!cleanText($linkTextClone.text())) {
                      $originalLink.addClass("sns-source-media-hidden");
                    } else {
                      $img.addClass("sns-source-media-hidden");
                    }
                  } else {
                    $img.addClass("sns-source-media-hidden");
                  }
                });
                $postContent.find("p").each(function() {
                  var $paragraph = $(this);
                  if ($paragraph.closest(".sns-media-grid").length) {
                    return;
                  }
                  var $paragraphClone = $paragraph.clone();
                  $paragraphClone.find(".sns-source-media-hidden, img, br").remove();
                  if (!cleanText($paragraphClone.text())) {
                    $paragraph.addClass("sns-source-media-hidden");
                  }
                });
                if ($grid.children(".sns-media-item").length) {
                  $postContent.prepend($grid);
                }
              }
            }
            if (imageCount > 0 && remainingText) {
              $postContent.find("p").each(function() {
                var $paragraph = $(this);
                var meaningfulSeen = false;
                $paragraph.contents().each(function() {
                  if (meaningfulSeen) {
                    return;
                  }
                  if (this.nodeType === 3) {
                    if (cleanText(this.nodeValue || "")) {
                      meaningfulSeen = true;
                    }
                    return;
                  }
                  if (this.nodeType !== 1) {
                    return;
                  }
                  var $node = $(this);
                  if ($node.is("br")) {
                    $node.remove();
                    return;
                  }
                  if ($node.hasClass("sns-source-media-hidden") || $node.find(".sns-source-media-hidden").length) {
                    return;
                  }
                  meaningfulSeen = true;
                });
              });
            }
            renderReplyPreview($post, $postContent, replyPayload && replyPayload.meta ? replyPayload.meta : null);
          }
          $body.children(".sns-meta, .sns-controls, .sns-menu-toggle, .sns-menu").remove();
          $container.children(".sns-mini-avatar").remove();
          if (!isOwn) {
            if (avatar) {
              $('<img class="sns-mini-avatar" alt="">').attr("src", avatar).attr("data-sns-profile-link", "1").attr("role", "link").attr("tabindex", "0").attr("title", "\u041e\u0442\u043a\u0440\u044b\u0442\u044c \u043f\u0440\u043e\u0444\u0438\u043b\u044c " + author).prependTo($container);
            } else {
              $('<span class="sns-mini-avatar sns-mini-avatar-empty"></span>').text((author.charAt(0) || "?").toUpperCase()).attr("data-sns-profile-link", "1").attr("role", "link").attr("tabindex", "0").attr("title", "\u041e\u0442\u043a\u0440\u044b\u0442\u044c \u043f\u0440\u043e\u0444\u0438\u043b\u044c " + author).prependTo($container);
            }
          }
          var $meta = $('<div class="sns-meta"></div>');
          if (!isOwn) {
            $('<span class="sns-author-name"></span>').text(author).attr("data-sns-profile-link", "1").attr("role", "link").attr("tabindex", "0").attr("title", "\u041e\u0442\u043a\u0440\u044b\u0442\u044c \u043f\u0440\u043e\u0444\u0438\u043b\u044c " + author).appendTo($meta);
          }
          var storyMeta = storyPayload && storyPayload.meta ? storyPayload.meta : storyMetaFromPost($post);
          if (storyMeta && storyMeta.time) {
            $('<span class="sns-time sns-story-time"></span>').text(storyMeta.time).appendTo($meta);
          }
          if ($meta.children().length) {
            $body.prepend($meta);
          }
          var $edit = findActionLink($post, "edit");
          var $delete = findActionLink($post, "delete");
          if (true) {
            var $controls = $('<div class="sns-controls"></div>');
            var $toggle = $("<button " + 'type="button" ' + 'class="sns-menu-toggle" ' + 'aria-label="\u0414\u0435\u0439\u0441\u0442\u0432\u0438\u044f">' + "\u2022\u2022\u2022" + "</button>");
            var $menu = $('<div class="sns-menu"></div>');
            var $replyButton = $("<button " + 'type="button" ' + 'class="sns-action-reply sns-direct-reply" ' + 'title="\u041e\u0442\u0432\u0435\u0442\u0438\u0442\u044c" ' + 'aria-label="\u041e\u0442\u0432\u0435\u0442\u0438\u0442\u044c">' + '<span class="sns-action-reply-icon" aria-hidden="true">' + '<svg viewBox="0 0 24 24">' + '<path d="M9 7l-5 5 5 5"></path>' + '<path d="M4 12h9.5c3.9 0 6.5 2.1 6.5 6"></path>' + "</svg>" + "</span>" + "</button>");
            if ($edit.length) {
              $menu.append("<button " + 'type="button" ' + 'class="sns-action-edit">' + "\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u0442\u044c" + "</button>");
            }
            if ($delete.length) {
              $menu.append("<button " + 'type="button" ' + 'class="sns-action-delete">' + "\u0423\u0434\u0430\u043b\u0438\u0442\u044c" + "</button>");
            }
            $controls.append($replyButton);
            $body.append($controls);
            if ($edit.length || $delete.length) {
              $post.addClass("sns-has-actions");
              $body.append($toggle, $menu);
            } else {
              $post.removeClass("sns-has-actions");
            }
          }
          this.__snsEnhancedContent = $postContent.get(0) || null;
        });
        var $empty = $("#sns-empty");
        if (!visibleCount) {
          if (!$empty.length) {
            $topic.append('<div id="sns-empty">' + "<b>\u0427\u0430\u0442 \u043f\u043e\u043a\u0430 \u043f\u0443\u0441\u0442.</b>" + "<span>\u041d\u0430\u043f\u0438\u0448\u0438 \u043f\u0435\u0440\u0432\u043e\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435.</span>" + "</div>");
          }
        } else {
          $empty.remove();
        }
        renderStoryDateSeparators();
        syncStoryDraftFromHistory();
      }
      function getNativeForm() {
        return $("#post").first();
      }
      function getNativeReply() {
        return $("#main-reply").first();
      }
      function hideNativeForm() {
        var $form = getNativeForm();
        if (!$form.length) {
          return;
        }
        $form.removeClass("sns-composer").addClass("sns-native-form");
      }
      function createComposer() {
        var $nativeForm = getNativeForm();
        var $nativeReply = getNativeReply();
        var $shell = $("#sns-chat-shell");
        if (!$nativeForm.length || !$nativeReply.length || !$shell.length) {
          return;
        }
        hideNativeForm();
        if ($("#sns-composer-ui").length) {
          return;
        }
        var $composer = $('<div id="sns-composer-ui"></div>');
        var $replyBar = $('<div id="sns-reply-compose" aria-hidden="true">' + '<span class="sns-reply-compose-copy">' + '<span class="sns-reply-compose-author"></span>' + '<span class="sns-reply-compose-text"></span>' + "</span>" + '<button type="button" class="sns-reply-compose-close" title="\u041e\u0442\u043c\u0435\u043d\u0438\u0442\u044c \u043e\u0442\u0432\u0435\u0442" aria-label="\u041e\u0442\u043c\u0435\u043d\u0438\u0442\u044c \u043e\u0442\u0432\u0435\u0442">\xd7</button>' + "</div>");
        var $plus = $("<button " + 'type="button" ' + 'class="sns-ui-plus" ' + 'title="\u0414\u043e\u0431\u0430\u0432\u0438\u0442\u044c">' + "+" + "</button>");
        var $formatButton = $("<button " + 'type="button" ' + 'class="sns-ui-format" ' + 'title="\u0424\u043e\u0440\u043c\u0430\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435" ' + 'aria-expanded="false">' + "Aa" + "</button>");
        var $storyButton = $("<button " + 'type="button" ' + 'class="sns-ui-storytime" ' + 'title="\u0421\u044e\u0436\u0435\u0442\u043d\u0430\u044f \u0434\u0430\u0442\u0430 / \u0432\u0440\u0435\u043c\u044f" ' + 'aria-expanded="false">' + '<svg viewBox="0 0 24 24" aria-hidden="true">' + '<rect x="3.5" y="5.5" width="17" height="15" rx="2.5"></rect>' + '<path d="M7.5 3.5v4"></path>' + '<path d="M16.5 3.5v4"></path>' + '<path d="M3.5 9.5h17"></path>' + '<circle cx="12" cy="14.5" r="2.5"></circle>' + '<path d="M12 13v1.7l1.2.8"></path>' + "</svg>" + "</button>");
        var $formatToolbar = $('<div id="sns-format-toolbar" aria-hidden="true">' + '<button type="button" class="sns-format-action sns-format-bold" data-sns-format="b" title="\u0416\u0438\u0440\u043d\u044b\u0439"><b>B</b></button>' + '<button type="button" class="sns-format-action sns-format-italic" data-sns-format="i" title="\u041a\u0443\u0440\u0441\u0438\u0432"><i>I</i></button>' + '<button type="button" class="sns-format-action sns-format-underline" data-sns-format="u" title="\u041f\u043e\u0434\u0447\u0435\u0440\u043a\u043d\u0443\u0442\u044b\u0439"><u>U</u></button>' + '<button type="button" class="sns-format-action sns-format-strike" data-sns-format="s" title="\u0417\u0430\u0447\u0435\u0440\u043a\u043d\u0443\u0442\u044b\u0439"><s>S</s></button>' + '<span class="sns-format-divider"></span>' + '<button type="button" class="sns-format-action sns-format-link" data-sns-format="url" title="\u0421\u0441\u044b\u043b\u043a\u0430">URL</button>' + "</div>");
        var $inputWrap = $('<div class="sns-ui-input-wrap"></div>');
        var $input = $("<textarea " + 'id="sns-ui-input" ' + 'rows="1" ' + 'placeholder="\u041d\u0430\u043f\u0438\u0441\u0430\u0442\u044c \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435...">' + "</textarea>");
        var $send = $("<button " + 'type="button" ' + 'class="sns-ui-send" ' + 'title="\u041e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c">' + "\u27a4" + "</button>");
        var $attachMenu = $('<div id="sns-attach-menu">' + "<button " + 'type="button" ' + 'class="sns-attach-photo">' + '<span class="sns-attach-icon sns-attach-icon-photo" aria-hidden="true">' + '<svg viewBox="0 0 24 24">' + '<path d="M5 3.5h14A2.5 2.5 0 0 1 21.5 6v12A2.5 2.5 0 0 1 19 20.5H5A2.5 2.5 0 0 1 2.5 18V6A2.5 2.5 0 0 1 5 3.5zm0 2A.5.5 0 0 0 4.5 6v9.15l4.2-4.2a1.2 1.2 0 0 1 1.7 0l2.05 2.05 3.15-3.15a1.2 1.2 0 0 1 1.7 0l2.2 2.2V6a.5.5 0 0 0-.5-.5H5zm3.2 1.65a2.15 2.15 0 1 1 0 4.3 2.15 2.15 0 0 1 0-4.3z"></path>' + "</svg>" + "</span>" + '<span class="sns-attach-label">\u0424\u043e\u0442\u043e</span>' + "</button>" + "<button " + 'type="button" ' + 'class="sns-attach-voice">' + '<span class="sns-attach-icon" aria-hidden="true">' + '<svg viewBox="0 0 24 24">' + '<rect x="9" y="3" width="6" height="11" rx="3"></rect>' + '<path d="M6 11a6 6 0 0 0 12 0"></path>' + '<path d="M12 17v4"></path>' + '<path d="M8.5 21h7"></path>' + "</svg>" + "</span>" + '<span class="sns-attach-label">\u0413\u043e\u043b\u043e\u0441\u043e\u0432\u043e\u0435</span>' + "</button>" + "<button " + 'type="button" ' + 'class="sns-attach-audio">' + '<span class="sns-attach-icon" aria-hidden="true">' + '<svg viewBox="0 0 24 24">' + '<path d="M9 18V5l10-2v13"></path>' + '<circle cx="6" cy="18" r="3"></circle>' + '<circle cx="16" cy="16" r="3"></circle>' + "</svg>" + "</span>" + '<span class="sns-attach-label">\u0410\u0443\u0434\u0438\u043e</span>' + "</button>" + "</div>");
        var $storyPanel = $('<div id="sns-story-compose" aria-hidden="true">' + '<div class="sns-story-compose-title">' + "<span>\u0421\u044e\u0436\u0435\u0442\u043d\u0430\u044f \u0434\u0430\u0442\u0430 \u0438 \u0432\u0440\u0435\u043c\u044f</span>" + '<button type="button" class="sns-story-compose-close" title="\u0417\u0430\u043a\u0440\u044b\u0442\u044c" aria-label="\u0417\u0430\u043a\u0440\u044b\u0442\u044c">\xd7</button>' + "</div>" + '<div class="sns-story-fields">' + '<label class="sns-story-field">' + "<span>\u0414\u0430\u0442\u0430 \u0441\u0446\u0435\u043d\u044b</span>" + '<input id="sns-story-date-input" type="text" autocomplete="off" placeholder="14 \u043e\u043a\u0442\u044f\u0431\u0440\u044f 2024">' + "</label>" + '<label class="sns-story-field">' + "<span>\u0412\u0440\u0435\u043c\u044f \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f</span>" + '<input id="sns-story-time-input" type="text" autocomplete="off" placeholder="23:40 \u0438\u043b\u0438 \u043d\u043e\u0447\u044c">' + "</label>" + "</div>" + '<div class="sns-story-compose-hint">' + "\u0414\u0430\u0442\u0430 \u043e\u0441\u0442\u0430\u0435\u0442\u0441\u044f \u0430\u043a\u0442\u0438\u0432\u043d\u043e\u0439 \u0434\u043b\u044f \u0441\u043b\u0435\u0434\u0443\u044e\u0449\u0438\u0445 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0439. \u0412\u0440\u0435\u043c\u044f \u0434\u0435\u0439\u0441\u0442\u0432\u0443\u0435\u0442 \u0442\u043e\u043b\u044c\u043a\u043e \u043d\u0430 \u0441\u043b\u0435\u0434\u0443\u044e\u0449\u0435\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435." + "</div>" + '<div class="sns-story-compose-actions">' + '<button type="button" class="sns-story-clear">\u0423\u0431\u0440\u0430\u0442\u044c \u0434\u0430\u0442\u0443</button>' + '<button type="button" class="sns-story-apply">\u0413\u043e\u0442\u043e\u0432\u043e</button>' + "</div>" + "</div>");
        var $voicePanel = $('<div id="sns-voice-compose" aria-hidden="true">' + '<div class="sns-voice-compose-title">' + '<svg viewBox="0 0 24 24" aria-hidden="true">' + '<path d="M12 2a3 3 0 0 0-3 3v7a3 3 0 0 0 6 0V5a3 3 0 0 0-3-3z"></path>' + '<path d="M5 10v2a7 7 0 0 0 14 0v-2"></path>' + '<path d="M12 19v3"></path>' + "</svg>" + "<span>\u0413\u043e\u043b\u043e\u0441\u043e\u0432\u043e\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435</span>" + "</div>" + '<textarea id="sns-voice-compose-text" rows="3" ' + 'placeholder="\u0427\u0442\u043e \u043f\u0435\u0440\u0441\u043e\u043d\u0430\u0436 \u0441\u043a\u0430\u0437\u0430\u043b \u0432 \u0433\u043e\u043b\u043e\u0441\u043e\u0432\u043e\u043c?"></textarea>' + '<div class="sns-voice-compose-foot">' + '<span class="sns-voice-compose-hint">\u0422\u0435\u043a\u0441\u0442 \u0431\u0443\u0434\u0435\u0442 \u0441\u043a\u0440\u044b\u0442 \u0434\u043e \u043d\u0430\u0436\u0430\u0442\u0438\u044f \u043d\u0430 \u0433\u043e\u043b\u043e\u0441\u043e\u0432\u043e\u0435</span>' + '<span class="sns-voice-compose-actions">' + '<button type="button" class="sns-voice-cancel">\u041e\u0442\u043c\u0435\u043d\u0430</button>' + '<button type="button" class="sns-voice-submit" disabled>\u041e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c</button>' + "</span>" + "</div>" + "</div>");
        var $audioPanel = $('<div id="sns-audio-compose" aria-hidden="true">' + '<div class="sns-audio-compose-title">' + '<svg viewBox="0 0 24 24" aria-hidden="true">' + '<path d="M9 18V5l10-2v13"></path>' + '<circle cx="6" cy="18" r="3"></circle>' + '<circle cx="16" cy="16" r="3"></circle>' + "</svg>" + "<span>\u0410\u0443\u0434\u0438\u043e / MP3</span>" + "</div>" + '<div class="sns-audio-upload-box">' + '<input class="sns-audio-file-input" id="sns-audio-file-input" type="file" accept=".mp3,.ogg,.oga,.wav,.m4a,.aac,.flac,audio/*">' + '<button type="button" class="sns-audio-file-pick">\u0417\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u0441 \u043a\u043e\u043c\u043f\u044c\u044e\u0442\u0435\u0440\u0430</button>' + '<div class="sns-audio-upload-info">' + '<span class="sns-audio-upload-status">\u0418\u043b\u0438 \u0432\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u043f\u0440\u044f\u043c\u0443\u044e \u0441\u0441\u044b\u043b\u043a\u0443 \u043d\u0438\u0436\u0435</span>' + '<span class="sns-audio-upload-progress"><i></i></span>' + "</div>" + "</div>" + '<div class="sns-audio-fields">' + '<div class="sns-audio-field is-wide">' + '<label for="sns-audio-url">\u041f\u0440\u044f\u043c\u0430\u044f \u0441\u0441\u044b\u043b\u043a\u0430 \u043d\u0430 \u0430\u0443\u0434\u0438\u043e *</label>' + '<input id="sns-audio-url" type="url" placeholder="https://site.com/track.mp3">' + "</div>" + '<div class="sns-audio-field">' + '<label for="sns-audio-title">\u041d\u0430\u0437\u0432\u0430\u043d\u0438\u0435</label>' + '<input id="sns-audio-title" type="text" placeholder="\u041d\u0430\u0437\u0432\u0430\u043d\u0438\u0435 \u0442\u0440\u0435\u043a\u0430">' + "</div>" + '<div class="sns-audio-field">' + '<label for="sns-audio-artist">\u0418\u0441\u043f\u043e\u043b\u043d\u0438\u0442\u0435\u043b\u044c</label>' + '<input id="sns-audio-artist" type="text" placeholder="\u0418\u0441\u043f\u043e\u043b\u043d\u0438\u0442\u0435\u043b\u044c">' + "</div>" + '<div class="sns-audio-field is-wide">' + '<label for="sns-audio-cover">\u041e\u0431\u043b\u043e\u0436\u043a\u0430 (\u0441\u0441\u044b\u043b\u043a\u0430, \u043d\u0435\u043e\u0431\u044f\u0437\u0430\u0442\u0435\u043b\u044c\u043d\u043e)</label>' + '<input id="sns-audio-cover" type="url" placeholder="https://site.com/cover.jpg">' + "</div>" + "</div>" + '<div class="sns-audio-compose-foot">' + '<span class="sns-audio-compose-hint">\u041d\u0443\u0436\u043d\u0430 \u043f\u0440\u044f\u043c\u0430\u044f https-\u0441\u0441\u044b\u043b\u043a\u0430 \u043d\u0430 mp3 / ogg / wav / m4a</span>' + '<span class="sns-audio-compose-actions">' + '<button type="button" class="sns-audio-cancel">\u041e\u0442\u043c\u0435\u043d\u0430</button>' + '<button type="button" class="sns-audio-submit" disabled>\u041e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c</button>' + "</span>" + "</div>" + "</div>");
        $inputWrap.append($input);
        $composer.append($replyBar, $plus, $formatButton, $storyButton, $inputWrap, $send, $formatToolbar, $storyPanel, $attachMenu, $voicePanel, $audioPanel);
        $shell.append($composer);
        updateReplyComposeBar();
        var $storyDateInput = $storyPanel.find("#sns-story-date-input");
        var $storyTimeInput = $storyPanel.find("#sns-story-time-input");
        function refreshStoryButtonState() {
          var active = snsStoryDateMode === 1 && !!snsStoryDate || !!snsStoryNextTime;
          $storyButton.toggleClass("is-active", active);
        }
        function closeStoryPanel() {
          $storyPanel.removeClass("is-open").attr("aria-hidden", "true");
          $storyButton.attr("aria-expanded", "false");
        }
        function openStoryPanel() {
          closeFormatToolbar();
          $storyDateInput.val(snsStoryDateMode === 1 ? snsStoryDate : "");
          $storyTimeInput.val(snsStoryNextTime);
          $storyPanel.addClass("is-open").attr("aria-hidden", "false");
          $storyButton.attr("aria-expanded", "true");
          setTimeout(function() {
            $storyDateInput.focus();
          }, 20);
        }
        $storyButton.on("click.snsStoryCompose", function(event) {
          event.preventDefault();
          event.stopPropagation();
          if ($storyPanel.hasClass("is-open")) {
            closeStoryPanel();
          } else {
            openStoryPanel();
          }
        });
        $storyPanel.on("click.snsStoryCompose", ".sns-story-compose-close", function(event) {
          event.preventDefault();
          closeStoryPanel();
          $input.focus();
        });
        $storyPanel.on("click.snsStoryCompose", ".sns-story-apply", function(event) {
          event.preventDefault();
          var date = String($storyDateInput.val() || "").replace(/\s+/g, " ").trim();
          var storyTime = String($storyTimeInput.val() || "").replace(/\s+/g, " ").trim();
          snsStoryUserTouched = true;
          if (date) {
            snsStoryDate = date;
            snsStoryDateMode = 1;
          } else if (snsStoryDateMode === 1 || snsStoryDate) {
            snsStoryDate = "";
            snsStoryDateMode = 2;
          }
          snsStoryNextTime = storyTime;
          refreshStoryButtonState();
          closeStoryPanel();
          $input.focus();
        });
        $storyPanel.on("click.snsStoryCompose", ".sns-story-clear", function(event) {
          event.preventDefault();
          snsStoryUserTouched = true;
          snsStoryDate = "";
          snsStoryDateMode = 2;
          snsStoryNextTime = "";
          $storyDateInput.val("");
          $storyTimeInput.val("");
          refreshStoryButtonState();
          closeStoryPanel();
          $input.focus();
        });
        $storyPanel.on("keydown.snsStoryCompose", "input", function(event) {
          if (event.key === "Escape") {
            event.preventDefault();
            closeStoryPanel();
            $input.focus();
          }
          if ((event.ctrlKey || event.metaKey) && event.key === "Enter") {
            event.preventDefault();
            $storyPanel.find(".sns-story-apply").trigger("click");
          }
        });
        $(document).off("mousedown.snsStoryCompose").on("mousedown.snsStoryCompose", function(event) {
          if (!$(event.target).closest("#sns-story-compose, .sns-ui-storytime").length) {
            closeStoryPanel();
          }
        });
        refreshStoryButtonState();
        $replyBar.on("click.snsReplyCompose", ".sns-reply-compose-close", function(event) {
          event.preventDefault();
          event.stopPropagation();
          clearReplyDraft();
          $input.focus();
        });
        function compactAttachMenuLayout() {
          var widths = [];
          if (window.visualViewport && isFinite(window.visualViewport.width)) {
            widths.push(window.visualViewport.width);
          }
          if (window.screen && isFinite(window.screen.width)) {
            widths.push(window.screen.width);
          }
          if (isFinite(window.innerWidth)) {
            widths.push(window.innerWidth);
          }
          var smallest = widths.length ? Math.min.apply(Math, widths) : window.innerWidth;
          return smallest <= 700 || window.matchMedia && window.matchMedia("(pointer: coarse) and (max-device-width: 900px)").matches;
        }
        function ensureAttachMenuHost() {
          if (compactAttachMenuLayout()) {
            if (!$attachMenu.parent().is($composer)) {
              $attachMenu.detach().appendTo($composer);
            }
            $attachMenu.removeClass("sns-attach-menu-portal").addClass("sns-attach-menu-mobile-inline");
            return;
          }
          if ($attachMenu.parent().get(0) !== document.body) {
            $attachMenu.detach().appendTo(document.body);
          }
          $attachMenu.removeClass("sns-attach-menu-mobile-inline").addClass("sns-attach-menu-portal");
        }
        ensureAttachMenuHost();
        function positionAttachMenu() {
          var plusNode = $plus.get(0);
          if (!plusNode) {
            return;
          }
          ensureAttachMenuHost();
          if ($attachMenu.hasClass("sns-attach-menu-mobile-inline")) {
            var plusPosition = $plus.position();
            var menuWidth = 145;
            var composerWidth = $composer.innerWidth() || 0;
            var localLeft = Math.max(8, Math.round(plusPosition.left));
            if (composerWidth && localLeft + menuWidth > composerWidth - 8) {
              localLeft = Math.max(8, composerWidth - menuWidth - 8);
            }
            $attachMenu.css({
              position: "absolute",
              left: localLeft + "px",
              top: "auto",
              right: "auto",
              bottom: "calc(100% + 9px)"
            });
            return;
          }
          var rect = plusNode.getBoundingClientRect();
          var menuWidth = 145;
          var left = Math.round(rect.left);
          if (left + menuWidth > window.innerWidth - 10) {
            left = Math.max(10, window.innerWidth - menuWidth - 10);
          }
          var menuHeight = 174;
          var top = Math.max(10, Math.round(rect.top - menuHeight - 10));
          $attachMenu.css({
            position: "fixed",
            left: left + "px",
            top: top + "px",
            right: "auto",
            bottom: "auto"
          });
        }
        function resizeInput() {
          $input.css("height", "40px");
          var height = Math.min(Math.max($input[0].scrollHeight, 40), 120);
          $input.css("height", height + "px");
        }
        $input.on("input.snsComposer", resizeInput);
        function updateSendState() {
          $send.prop("disabled", !$.trim($input.val()));
        }
        $input.on("input.snsState", updateSendState);
        updateSendState();
        function nativeSubmitButton() {
          var $button = $nativeForm.find('input[type="submit"][name="submit"],' + 'button[type="submit"][name="submit"]').first();
          if (!$button.length) {
            $button = $nativeForm.find('input[type="submit"],' + 'button[type="submit"]').first();
          }
          return $button;
        }
        var snsPendingCounter = 0;
        function latestOwnRealPostId() {
          var me = String(getCurrentUser() || "").toLowerCase();
          var maxId = 0;
          $("#sns-chat-shell>.topic .post[id]").each(function() {
            var $post = $(this);
            if (me && String(getPostAuthor($post) || "").toLowerCase() !== me) {
              return;
            }
            var id = parseInt(String($post.attr("id") || "").replace(/^p/i, "").replace(/\D+/g, ""), 10);
            if (id && id > maxId) {
              maxId = id;
            }
          });
          return maxId;
        }
        function latestTopicRealPostId() {
          var maxId = 0;
          $("#sns-chat-shell>.topic .post[id]").each(function() {
            var id = parseInt(String($(this).attr("id") || "").replace(/^p/i, "").replace(/\D+/g, ""), 10);
            if (id && id > maxId) {
              maxId = id;
            }
          });
          return maxId;
        }
        function pendingPreviewText(value) {
          var result = String(value || "");
          var hadImage = /\[img(?:=[^\]]*)?\][\s\S]*?\[\/img\]/i.test(result) || /<img\b/i.test(result);
          result = result.replace(/\[img(?:=[^\]]*)?\][\s\S]*?\[\/img\]/gi, "").replace(/<img\b[^>]*>/gi, "").replace(/\[\/?(?:b|i|u|s)\]/gi, "").replace(/\[url(?:=[^\]]*)?\]/gi, "").replace(/\[\/url\]/gi, "").replace(/<[^>]+>/g, "").trim();
          if (!result && hadImage) {
            result = "\u0438\u0437\u043e\u0431\u0440\u0430\u0436\u0435\u043d\u0438\u0435";
          }
          return result || "\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435";
        }
        function pendingStack() {
          var $topic = $("#sns-chat-shell>.topic").first();
          if (!$topic.length) {
            return $();
          }
          var $stack = $topic.children("#sns-pending-stack").first();
          if (!$stack.length) {
            $stack = $('<div id="sns-pending-stack"></div>');
            $topic.append($stack);
          }
          return $stack;
        }
        function scrollPendingIntoView() {
          var topicNode = $("#sns-chat-shell>.topic").get(0);
          if (!topicNode) {
            return;
          }
          topicNode.scrollTop = topicNode.scrollHeight;
        }
        function renderPendingStickerMessage($row, rawValue) {
          var raw = String(rawValue || "");
          var stickerRx = /\[img=smalimg\]([\s\S]*?)\[\/img\]/gi;
          var matches = [];
          var match;
          while (match = stickerRx.exec(raw)) {
            matches.push({
              full: match[0],
              url: String(match[1] || "").trim(),
              index: match.index
            });
          }
          if (!matches.length) {
            return false;
          }
          var textOnly = raw.replace(stickerRx, "").replace(/\[\/?(?:b|i|u|s)\]/gi, "").replace(/\[url(?:=[^\]]*)?\]/gi, "").replace(/\[\/url\]/gi, "").trim();
          var hasText = !!textOnly;
          $row.addClass(hasText ? "sns-pending-sticker-text" : "sns-pending-sticker-only");
          var $bubble = $row.find(".sns-pending-bubble").empty();
          var cursor = 0;
          matches.forEach(function(entry) {
            var before = raw.slice(cursor, entry.index);
            before = before.replace(/\[\/?(?:b|i|u|s)\]/gi, "").replace(/\[url(?:=[^\]]*)?\]/gi, "").replace(/\[\/url\]/gi, "").replace(/\s+/g, " ");
            if (before) {
              $bubble.append(document.createTextNode(before));
            }
            if (entry.url) {
              $('<img class="sns-pending-sticker" alt="">').attr("src", entry.url).appendTo($bubble);
            }
            cursor = entry.index + entry.full.length;
          });
          var after = raw.slice(cursor).replace(/\[\/?(?:b|i|u|s)\]/gi, "").replace(/\[url(?:=[^\]]*)?\]/gi, "").replace(/\[\/url\]/gi, "").replace(/\s+/g, " ");
          if (after) {
            $bubble.append(document.createTextNode(after));
          }
          return true;
        }
        function appendPendingBubble(item) {
          if (!item) {
            return;
          }
          var $stack = pendingStack();
          if (!$stack.length) {
            return;
          }
          var $row = $('<div class="sns-pending-row">' + '<div class="sns-pending-meta"></div>' + '<div class="sns-pending-bubble"></div>' + "</div>").attr("data-sns-client-id", item.clientId);
          $row.find(".sns-pending-meta").text("\u0432 \u043e\u0447\u0435\u0440\u0435\u0434\u0438\u2026");
          var pendingStory = parseStoryWrappedRaw(item.value);
          var pendingRawValue = pendingStory ? pendingStory.body : item.value;
          var pendingReply = parseReplyWrappedRaw(pendingRawValue);
          var pendingBodyValue = pendingReply ? pendingReply.body : pendingRawValue;
          var pendingAudio = parseAudioMarker(pendingBodyValue);
          var pendingVoice = parseVoiceMarker(pendingBodyValue);
          if (pendingAudio) {
            $row.addClass("sns-pending-audio");
            $row.find(".sns-pending-bubble").empty().append(createAudioCard(pendingAudio.data));
          } else if (pendingVoice) {
            $row.addClass("sns-pending-voice");
            $row.find(".sns-pending-bubble").empty().append(createVoiceCard(pendingVoice.text, false));
          } else {
            if (!renderPendingStickerMessage($row, pendingBodyValue)) {
              $row.find(".sns-pending-bubble").text(pendingPreviewText(pendingBodyValue));
            }
          }
          if (pendingReply && pendingReply.meta) {
            $row.find(".sns-pending-bubble").prepend(createReplyPreview(pendingReply.meta));
          }
          $stack.append($row);
          item.$pending = $row;
          scrollPendingIntoView();
        }
        function setPendingState(item, state) {
          if (!item || !item.$pending || !item.$pending.length) {
            return;
          }
          var label = state === "sending" ? "\u043e\u0442\u043f\u0440\u0430\u0432\u043b\u044f\u0435\u0442\u0441\u044f\u2026" : "\u0432 \u043e\u0447\u0435\u0440\u0435\u0434\u0438\u2026";
          item.$pending.toggleClass("is-sending", state === "sending").find(".sns-pending-meta").text(label);
        }
        function removePendingBubble(item) {
          if (!item || !item.$pending || !item.$pending.length) {
            return;
          }
          var $row = item.$pending;
          item.$pending = null;
          $row.addClass("is-confirmed");
          setTimeout(function() {
            $row.remove();
            var $stack = $("#sns-pending-stack");
            if ($stack.length && !$stack.children().length) {
              $stack.remove();
            }
          }, 160);
        }
        var snsSendQueue = [];
        var snsSendActive = null;
        var snsLastConfirmedAt = 0;
        var snsSendQueueTimer = null;
        var SNS_SEND_MIN_GAP = 5500;
        function clearQueueTimer() {
          if (snsSendQueueTimer) {
            clearTimeout(snsSendQueueTimer);
            snsSendQueueTimer = null;
          }
        }
        function processSendQueue() {
          clearQueueTimer();
          if (snsSendActive || !snsSendQueue.length) {
            return;
          }
          var elapsed = Date.now() - snsLastConfirmedAt;
          var wait = snsLastConfirmedAt ? Math.max(0, SNS_SEND_MIN_GAP - elapsed) : 0;
          if (wait > 0) {
            snsSendQueueTimer = setTimeout(processSendQueue, wait);
            return;
          }
          var item = snsSendQueue.shift();
          if (!item || !$.trim(item.value)) {
            processSendQueue();
            return;
          }
          var $nativeSubmit = nativeSubmitButton();
          if (!$nativeSubmit.length) {
            console.warn("SNS: \u043d\u0435 \u043d\u0430\u0439\u0434\u0435\u043d\u0430 \u043a\u043d\u043e\u043f\u043a\u0430 \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0438 RusFF");
            setPendingState(item, "queued");
            snsSendQueue.unshift(item);
            snsSendQueueTimer = setTimeout(processSendQueue, 1200);
            return;
          }
          snsSendActive = item;
          setPendingState(item, "sending");
          $nativeReply.val(item.value).trigger("input").trigger("change");
          item.nativeBaselineTopicPostId = latestTopicRealPostId();
          $nativeSubmit.trigger("click");
        }
        function queueMessage() {
          var value = String($input.val() || "");
          if (!$.trim(value)) {
            return;
          }
          if (snsReplyDraft) {
            value = wrapReplyMessage(snsReplyDraft, value);
          }
          var storyMeta = storyMetaForNextMessage();
          if (storyMeta) {
            value = wrapStoryMessage(storyMeta, value);
          }
          var item = {
            value: value,
            queuedAt: Date.now(),
            baselineOwnPostId: latestOwnRealPostId(),
            clientId: "snsq_" + Date.now() + "_" + ++snsPendingCounter,
            $pending: null
          };
          snsSendQueue.push(item);
          snsStoryNextTime = "";
          if (snsStoryDateMode === 2) {
            snsStoryDateMode = 0;
          }
          refreshStoryButtonState();
          $storyTimeInput.val("");
          appendPendingBubble(item);
          clearReplyDraft();
          $input.val("").trigger("input");
          closeFormatToolbar();
          processSendQueue();
        }
        function confirmQueuedSend() {
          if (!snsSendActive) {
            return false;
          }
          var confirmedItem = snsSendActive;
          snsSendActive = null;
          snsLastConfirmedAt = Date.now();
          window.__SNS_LAST_CONFIRMED_TOPIC_BASELINE__ = parseInt(confirmedItem.nativeBaselineTopicPostId, 10) || 0;
          if (confirmedItem.$pending && confirmedItem.$pending.length) {
            confirmedItem.$pending.removeClass("is-sending").addClass("is-server-confirmed").find(".sns-pending-meta").text("").hide();
          }
          [ 20, 80, 220, 700, 1800 ].forEach(function(delay) {
            setTimeout(function() {
              if (!confirmedItem.$pending || !confirmedItem.$pending.length) {
                return;
              }
              if (latestOwnRealPostId() > (parseInt(confirmedItem.baselineOwnPostId, 10) || 0)) {
                removePendingBubble(confirmedItem);
              }
            }, delay);
          });
          $nativeReply.val("");
          processSendQueue();
          return true;
        }
        window.SNSConfirmQueuedSend = confirmQueuedSend;
        window.SNSProcessSendQueue = processSendQueue;
        $send.on("click.snsComposer", function() {
          queueMessage();
        });
        function closeFormatToolbar() {
          $formatToolbar.removeClass("is-open").attr("aria-hidden", "true");
          $formatButton.removeClass("is-active").attr("aria-expanded", "false");
        }
        function openFormatToolbar() {
          $formatToolbar.addClass("is-open").attr("aria-hidden", "false");
          $formatButton.addClass("is-active").attr("aria-expanded", "true");
        }
        function insertAroundSelection(openTag, closeTag, placeholder) {
          var node = $input.get(0);
          if (!node) {
            return;
          }
          var value = String($input.val() || "");
          var start = typeof node.selectionStart === "number" ? node.selectionStart : value.length;
          var end = typeof node.selectionEnd === "number" ? node.selectionEnd : start;
          var selected = value.slice(start, end);
          var inner = selected || String(placeholder || "");
          var replacement = openTag + inner + closeTag;
          var nextValue = value.slice(0, start) + replacement + value.slice(end);
          $input.val(nextValue).trigger("input");
          setTimeout(function() {
            var liveNode = $input.get(0);
            if (!liveNode) {
              return;
            }
            $input.focus();
            var contentStart = start + openTag.length;
            if (selected) {
              var caret = start + replacement.length;
              try {
                liveNode.setSelectionRange(caret, caret);
              } catch (error) {}
            } else if (inner) {
              try {
                liveNode.setSelectionRange(contentStart, contentStart + inner.length);
              } catch (error) {}
            } else {
              try {
                liveNode.setSelectionRange(contentStart, contentStart);
              } catch (error) {}
            }
          }, 0);
        }
        function insertUrl() {
          var node = $input.get(0);
          if (!node) {
            return;
          }
          var value = String($input.val() || "");
          var start = typeof node.selectionStart === "number" ? node.selectionStart : value.length;
          var end = typeof node.selectionEnd === "number" ? node.selectionEnd : start;
          var selected = value.slice(start, end);
          if (selected && /^(?:https?:\/\/|www\.)/i.test(selected)) {
            insertAroundSelection("[url]", "[/url]", "");
            return;
          }
          if (selected) {
            var replacement = "[url=]" + selected + "[/url]";
            var nextValue = value.slice(0, start) + replacement + value.slice(end);
            $input.val(nextValue).trigger("input");
            setTimeout(function() {
              var liveNode = $input.get(0);
              if (!liveNode) {
                return;
              }
              $input.focus();
              var caret = start + "[url=".length;
              try {
                liveNode.setSelectionRange(caret, caret);
              } catch (error) {}
            }, 0);
            return;
          }
          insertAroundSelection("[url]", "[/url]", "https://");
        }
        $formatButton.on("click.snsFormat", function(event) {
          event.preventDefault();
          event.stopPropagation();
          if ($formatButton.prop("disabled")) {
            return;
          }
          if ($formatToolbar.hasClass("is-open")) {
            closeFormatToolbar();
          } else {
            $attachMenu.removeClass("is-open");
            closeVoicePanel(false);
            closeAudioPanel(false);
            $plus.removeClass("is-active");
            openFormatToolbar();
          }
        });
        $formatToolbar.on("click.snsFormat", ".sns-format-action", function(event) {
          event.preventDefault();
          event.stopPropagation();
          var type = String($(this).attr("data-sns-format") || "");
          if (type === "url") {
            insertUrl();
            return;
          }
          var tags = {
            b: [ "[b]", "[/b]" ],
            i: [ "[i]", "[/i]" ],
            u: [ "[u]", "[/u]" ],
            s: [ "[s]", "[/s]" ]
          };
          if (!tags[type]) {
            return;
          }
          insertAroundSelection(tags[type][0], tags[type][1], "");
        });
        $(document).off("click.snsFormatClose").on("click.snsFormatClose", function(event) {
          if (!$(event.target).closest("#sns-format-toolbar, .sns-ui-format").length) {
            closeFormatToolbar();
          }
        }).off("keydown.snsFormatClose").on("keydown.snsFormatClose", function(event) {
          if (event.key === "Escape") {
            closeFormatToolbar();
          }
        });
        $plus.on("click.snsAttach", function(event) {
          event.preventDefault();
          event.stopPropagation();
          var $imageArea = $("#image-area.sns-image-area");
          if ($imageArea.length && $imageArea.is(":visible")) {
            closeImageArea();
            return;
          }
          closeFormatToolbar();
          closeVoicePanel(false);
          closeAudioPanel(false);
          var willOpen = !$attachMenu.hasClass("is-open");
          $plus.toggleClass("is-active", willOpen);
          if (willOpen) {
            positionAttachMenu();
            $attachMenu.addClass("is-open");
          } else {
            $attachMenu.removeClass("is-open");
          }
        });
        var photoSyncTimer = null;
        function syncNativeToSNS() {
          var nativeValue = $nativeReply.val();
          if (typeof nativeValue === "string" && nativeValue !== $input.val()) {
            $input.val(nativeValue).trigger("input");
          }
        }
        function stopPhotoSync() {
          if (photoSyncTimer) {
            clearInterval(photoSyncTimer);
            photoSyncTimer = null;
          }
        }
        function ensureImageAreaClose($area) {
          if (!$area || !$area.length) {
            return;
          }
          if (!$area.children(".sns-image-area-close").length) {
            $area.prepend('<button type="button" class="sns-image-area-close">&times;</button>');
          }
        }
        function moveImageAreaIntoSNS() {
          var $area = $("#image-area").first();
          if (!$area.length) {
            return $area;
          }
          if (!$area.parent().is($shell)) {
            $area.detach().appendTo($shell);
          }
          $area.addClass("sns-image-area");
          ensureImageAreaClose($area);
          return $area;
        }
        function closeImageArea() {
          stopPhotoSync();
          $("#image-area.sns-image-area").hide();
          $plus.removeClass("is-active");
          $attachMenu.removeClass("is-open");
          syncNativeToSNS();
          setTimeout(function() {
            $input.focus();
          }, 20);
        }
        function startPhotoSync() {
          stopPhotoSync();
          var tries = 0;
          photoSyncTimer = setInterval(function() {
            tries++;
            moveImageAreaIntoSNS();
            syncNativeToSNS();
            if (tries > 250) {
              stopPhotoSync();
            }
          }, 120);
        }
        function openNativeImagePicker() {
          $nativeReply.val($input.val()).trigger("input").trigger("change");
          var $imageButton = $("#button-image").first();
          if (!$imageButton.length) {
            console.warn("SNS: #button-image not found");
            return;
          }
          moveImageAreaIntoSNS();
          var $clickTarget = $imageButton.find("img").first();
          if ($clickTarget.length) {
            $clickTarget.trigger("click");
          } else {
            $imageButton.trigger("click");
          }
          setTimeout(function() {
            moveImageAreaIntoSNS();
            startPhotoSync();
          }, 30);
        }
        $(window).off("resize.snsAttachPortal scroll.snsAttachPortal").on("resize.snsAttachPortal scroll.snsAttachPortal", function() {
          if ($attachMenu.hasClass("is-open")) {
            positionAttachMenu();
          }
        });
        var $audioFileInput = $audioPanel.find("#sns-audio-file-input");
        var $audioFilePick = $audioPanel.find(".sns-audio-file-pick");
        var $audioUploadStatus = $audioPanel.find(".sns-audio-upload-status");
        var $audioUploadProgress = $audioPanel.find(".sns-audio-upload-progress");
        var audioUploadXhr = null;
        var audioUploadBusy = false;
        var $audioUrl = $audioPanel.find("#sns-audio-url");
        var $audioTitle = $audioPanel.find("#sns-audio-title");
        var $audioArtist = $audioPanel.find("#sns-audio-artist");
        var $audioCover = $audioPanel.find("#sns-audio-cover");
        var $audioSubmit = $audioPanel.find(".sns-audio-submit");
        function setAudioUploadStatus(textValue, state, percent) {
          $audioUploadStatus.removeClass("is-error is-success").toggleClass("is-error", state === "error").toggleClass("is-success", state === "success").text(textValue);
          var showProgress = state === "uploading";
          $audioUploadProgress.toggleClass("is-visible", showProgress);
          $audioUploadProgress.find("i").css("width", Math.max(0, Math.min(100, Number(percent) || 0)) + "%");
        }
        function refreshAudioUploadAvailability() {
          var cfg = cloudinaryAudioConfig();
          var ready = !!(cfg.cloudName && cfg.uploadPreset);
          $audioFilePick.prop("disabled", !ready || audioUploadBusy).attr("title", ready ? "\u041c\u0430\u043a\u0441. " + cfg.maxMb + " MB" : "\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u0441 \u043a\u043e\u043c\u043f\u044c\u044e\u0442\u0435\u0440\u0430 \u0435\u0449\u0435 \u043d\u0435 \u043d\u0430\u0441\u0442\u0440\u043e\u0435\u043d\u0430");
          if (!ready && !audioUploadBusy) {
            setAudioUploadStatus("\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u0441 \u043a\u043e\u043c\u043f\u044c\u044e\u0442\u0435\u0440\u0430 \u043d\u0435 \u043d\u0430\u0441\u0442\u0440\u043e\u0435\u043d\u0430 \u0430\u0434\u043c\u0438\u043d\u043e\u043c; \u0441\u0441\u044b\u043b\u043a\u0443 \u043c\u043e\u0436\u043d\u043e \u0432\u0441\u0442\u0430\u0432\u0438\u0442\u044c \u0432\u0440\u0443\u0447\u043d\u0443\u044e", "", 0);
          }
        }
        function abortAudioUpload() {
          if (audioUploadXhr && audioUploadBusy) {
            try {
              audioUploadXhr.abort();
            } catch (error) {}
          }
          audioUploadXhr = null;
          audioUploadBusy = false;
        }
        function uploadAudioFileToCloudinary(file) {
          var deferred = $.Deferred();
          var cfg = cloudinaryAudioConfig();
          if (!cfg.cloudName || !cfg.uploadPreset) {
            deferred.reject("\u0410\u0434\u043c\u0438\u043d \u0435\u0449\u0435 \u043d\u0435 \u043d\u0430\u0441\u0442\u0440\u043e\u0438\u043b Cloudinary");
            return deferred.promise();
          }
          if (!audioUploadFileAllowed(file)) {
            deferred.reject("\u041d\u0443\u0436\u0435\u043d \u0430\u0443\u0434\u0438\u043e\u0444\u0430\u0439\u043b: mp3 / ogg / wav / m4a / aac / flac");
            return deferred.promise();
          }
          var maxBytes = cfg.maxMb * 1024 * 1024;
          if (Number(file.size) > maxBytes) {
            deferred.reject("\u0424\u0430\u0439\u043b \u0431\u043e\u043b\u044c\u0448\u0435 " + cfg.maxMb + " MB");
            return deferred.promise();
          }
          var endpoint = "https://api.cloudinary.com/v1_1/" + encodeURIComponent(cfg.cloudName) + "/video/upload";
          var formData = new FormData;
          formData.append("file", file);
          formData.append("upload_preset", cfg.uploadPreset);
          var xhr = new XMLHttpRequest;
          audioUploadXhr = xhr;
          audioUploadBusy = true;
          refreshAudioUploadAvailability();
          setAudioUploadStatus("\u0417\u0430\u0433\u0440\u0443\u0436\u0430\u044e " + String(file.name || "\u0430\u0443\u0434\u0438\u043e") + "\u2026", "uploading", 0);
          xhr.open("POST", endpoint, true);
          xhr.upload.onprogress = function(event) {
            if (!event.lengthComputable) {
              return;
            }
            setAudioUploadStatus("\u0417\u0430\u0433\u0440\u0443\u0436\u0430\u044e " + String(file.name || "\u0430\u0443\u0434\u0438\u043e") + "\u2026 " + Math.round(event.loaded / event.total * 100) + "%", "uploading", event.loaded / event.total * 100);
          };
          xhr.onload = function() {
            audioUploadBusy = false;
            audioUploadXhr = null;
            var response = null;
            try {
              response = JSON.parse(xhr.responseText || "{}");
            } catch (error) {}
            if (xhr.status >= 200 && xhr.status < 300 && response && response.secure_url) {
              deferred.resolve(response);
            } else {
              var message = response && response.error && response.error.message ? response.error.message : "\u041e\u0448\u0438\u0431\u043a\u0430 Cloudinary (" + xhr.status + ")";
              deferred.reject(message);
            }
            refreshAudioUploadAvailability();
          };
          xhr.onerror = function() {
            audioUploadBusy = false;
            audioUploadXhr = null;
            refreshAudioUploadAvailability();
            deferred.reject("\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u0444\u0430\u0439\u043b");
          };
          xhr.onabort = function() {
            audioUploadBusy = false;
            audioUploadXhr = null;
            refreshAudioUploadAvailability();
            deferred.reject("\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u043e\u0442\u043c\u0435\u043d\u0435\u043d\u0430");
          };
          xhr.send(formData);
          return deferred.promise();
        }
        function updateAudioSubmitState() {
          var ok = !!audioSafeUrl($audioUrl.val());
          $audioSubmit.prop("disabled", !ok || audioUploadBusy);
          $audioUrl.toggleClass("is-invalid", !!$.trim($audioUrl.val()) && !ok);
        }
        function closeAudioPanel(clearValue) {
          if (clearValue) {
            abortAudioUpload();
          }
          $audioPanel.removeClass("is-open").attr("aria-hidden", "true");
          if (clearValue) {
            $audioPanel.find("input").val("").removeClass("is-invalid");
            $audioFileInput.val("");
            setAudioUploadStatus(cloudinaryAudioReady() ? "\u0418\u043b\u0438 \u0432\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u043f\u0440\u044f\u043c\u0443\u044e \u0441\u0441\u044b\u043b\u043a\u0443 \u043d\u0438\u0436\u0435" : "\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u0441 \u043a\u043e\u043c\u043f\u044c\u044e\u0442\u0435\u0440\u0430 \u043d\u0435 \u043d\u0430\u0441\u0442\u0440\u043e\u0435\u043d\u0430 \u0430\u0434\u043c\u0438\u043d\u043e\u043c; \u0441\u0441\u044b\u043b\u043a\u0443 \u043c\u043e\u0436\u043d\u043e \u0432\u0441\u0442\u0430\u0432\u0438\u0442\u044c \u0432\u0440\u0443\u0447\u043d\u0443\u044e", "", 0);
          }
          updateAudioSubmitState();
          $plus.removeClass("is-active");
        }
        function openAudioPanel() {
          refreshAudioUploadAvailability();
          closeFormatToolbar();
          closeVoicePanel(false);
          $attachMenu.removeClass("is-open");
          $plus.removeClass("is-active");
          $audioPanel.addClass("is-open").attr("aria-hidden", "false");
          setTimeout(function() {
            $audioUrl.focus();
          }, 20);
        }
        $audioFilePick.on("click.snsAudioUpload", function(event) {
          event.preventDefault();
          event.stopPropagation();
          if (!cloudinaryAudioReady() || audioUploadBusy) {
            return;
          }
          $audioFileInput.trigger("click");
        });
        $audioFileInput.on("change.snsAudioUpload", function() {
          var file = this.files && this.files[0] ? this.files[0] : null;
          if (!file) {
            return;
          }
          if (!$audioTitle.val()) {
            $audioTitle.val(audioTitleFromFileName(file.name));
          }
          uploadAudioFileToCloudinary(file).done(function(response) {
            var secureUrl = String(response.secure_url || "");
            $audioUrl.val(secureUrl).removeClass("is-invalid");
            if (!$audioTitle.val() && response.original_filename) {
              $audioTitle.val(audioTitleFromFileName(response.original_filename));
            }
            setAudioUploadStatus("\u0413\u043e\u0442\u043e\u0432\u043e \u2014 \u0444\u0430\u0439\u043b \u0437\u0430\u0433\u0440\u0443\u0436\u0435\u043d", "success", 100);
            updateAudioSubmitState();
          }).fail(function(message) {
            if (String(message || "") === "\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u043e\u0442\u043c\u0435\u043d\u0435\u043d\u0430") {
              return;
            }
            setAudioUploadStatus(String(message || "\u041e\u0448\u0438\u0431\u043a\u0430 \u0437\u0430\u0433\u0440\u0443\u0437\u043a\u0438"), "error", 0);
            updateAudioSubmitState();
          });
        });
        $audioPanel.on("input.snsAudioCompose", "input", updateAudioSubmitState);
        $audioPanel.on("click.snsAudioCompose", ".sns-audio-cancel", function(event) {
          event.preventDefault();
          event.stopPropagation();
          closeAudioPanel(true);
          $input.focus();
        });
        $audioPanel.on("click.snsAudioCompose", ".sns-audio-submit", function(event) {
          event.preventDefault();
          event.stopPropagation();
          if (audioUploadBusy) {
            return;
          }
          var data = {
            url: $audioUrl.val(),
            title: $audioTitle.val(),
            artist: $audioArtist.val(),
            cover: $audioCover.val()
          };
          var marker = audioMarkerFromData(data);
          if (!marker) {
            $audioUrl.addClass("is-invalid").focus();
            return;
          }
          closeAudioPanel(true);
          $input.val(marker).trigger("input");
          queueMessage();
        });
        $audioPanel.on("keydown.snsAudioCompose", "input", function(event) {
          if (event.key === "Escape") {
            event.preventDefault();
            closeAudioPanel(false);
            $input.focus();
          }
          if ((event.ctrlKey || event.metaKey) && event.key === "Enter") {
            event.preventDefault();
            $audioSubmit.trigger("click");
          }
        });
        $attachMenu.on("click.snsAttach", ".sns-attach-audio", function(event) {
          event.preventDefault();
          event.stopPropagation();
          openAudioPanel();
        });
        var $voiceText = $voicePanel.find("#sns-voice-compose-text");
        var $voiceSubmit = $voicePanel.find(".sns-voice-submit");
        function closeVoicePanel(clearValue) {
          $voicePanel.removeClass("is-open").attr("aria-hidden", "true");
          if (clearValue) {
            $voiceText.val("");
          }
          $voiceSubmit.prop("disabled", !$.trim($voiceText.val()));
          $plus.removeClass("is-active");
        }
        function openVoicePanel() {
          closeFormatToolbar();
          closeAudioPanel(false);
          $attachMenu.removeClass("is-open");
          $plus.removeClass("is-active");
          $voicePanel.addClass("is-open").attr("aria-hidden", "false");
          setTimeout(function() {
            $voiceText.focus();
          }, 20);
        }
        $voiceText.on("input.snsVoiceCompose", function() {
          $voiceSubmit.prop("disabled", !$.trim($voiceText.val()));
        });
        $voicePanel.on("click.snsVoiceCompose", ".sns-voice-cancel", function(event) {
          event.preventDefault();
          event.stopPropagation();
          closeVoicePanel(true);
          $input.focus();
        });
        $voicePanel.on("click.snsVoiceCompose", ".sns-voice-submit", function(event) {
          event.preventDefault();
          event.stopPropagation();
          var spokenText = String($voiceText.val() || "").trim();
          if (!spokenText) {
            return;
          }
          var marker = voiceMarkerFromText(spokenText);
          if (!marker) {
            return;
          }
          closeVoicePanel(true);
          $input.val(marker).trigger("input");
          queueMessage();
        });
        $voiceText.on("keydown.snsVoiceCompose", function(event) {
          if (event.key === "Escape") {
            event.preventDefault();
            closeVoicePanel(false);
            $input.focus();
          }
          if ((event.ctrlKey || event.metaKey) && event.key === "Enter") {
            event.preventDefault();
            $voiceSubmit.trigger("click");
          }
        });
        $attachMenu.on("click.snsAttach", ".sns-attach-voice", function(event) {
          event.preventDefault();
          event.stopPropagation();
          openVoicePanel();
        });
        $attachMenu.on("click.snsAttach", ".sns-attach-photo", function(event) {
          event.preventDefault();
          event.stopPropagation();
          $attachMenu.removeClass("is-open");
          $plus.removeClass("is-active");
          openNativeImagePicker();
        });
        $(document).on("click.snsImageSync change.snsImageSync input.snsImageSync", "#image-area, #image-area *", function() {
          setTimeout(syncNativeToSNS, 40);
          setTimeout(syncNativeToSNS, 220);
        });
        $(document).off(".snsImageAreaClose").on("click.snsImageAreaClose", ".sns-image-area-close", function(event) {
          event.preventDefault();
          event.stopPropagation();
          closeImageArea();
        }).on("keydown.snsImageAreaClose", function(event) {
          if (event.key === "Escape" && $("#image-area.sns-image-area").is(":visible")) {
            closeImageArea();
          }
        });
        $(document).on("click.snsAttachClose", function(event) {
          if (!$(event.target).closest("#sns-attach-menu, .sns-ui-plus").length) {
            $attachMenu.removeClass("is-open");
            $plus.removeClass("is-active");
          }
        });
      }
      function updateAudioCard($audio) {
        if (document.hidden) return;
        if (!$audio || !$audio.length) {
          return;
        }
        var node = $audio.get(0);
        var $card = $audio.closest(".sns-audio-card");
        var duration = Number(node.duration);
        var current = Number(node.currentTime);
        var percent = isFinite(duration) && duration > 0 ? Math.max(0, Math.min(100, current / duration * 100)) : 0;
        $card.find(".sns-audio-progress").css("width", percent + "%");
        $card.find(".sns-audio-track").attr("aria-valuemin", "0").attr("aria-valuemax", isFinite(duration) ? String(Math.max(0, Math.floor(duration))) : "0").attr("aria-valuenow", String(Math.max(0, Math.floor(current))));
        $card.find(".sns-audio-time").text(isFinite(duration) && duration > 0 ? formatAudioClock(current) + " / " + formatAudioClock(duration) : "0:00");
        $card.find(".sns-audio-play").toggleClass("is-playing", !node.paused && !node.ended);
      }
      $(document).off(".snsAudioPlayer").on("click.snsAudioPlayer", ".sns-audio-play", function(event) {
        event.preventDefault();
        event.stopPropagation();
        var $card = $(this).closest(".sns-audio-card");
        var $audio = $card.find(".sns-audio-native").first();
        var node = $audio.get(0);
        if (!node) {
          return;
        }
        $(".sns-audio-native").not(node).each(function() {
          try {
            stopAudioProgressTicker(this);
            this.pause();
          } catch (error) {}
        });
        if (node.paused) {
          var playPromise = node.play();
          if (playPromise && typeof playPromise.catch === "function") {
            playPromise.catch(function() {
              $card.addClass("has-error");
            });
          }
        } else {
          node.pause();
        }
        updateAudioCard($audio);
      }).on("click.snsAudioPlayer", ".sns-audio-track", function(event) {
        event.preventDefault();
        event.stopPropagation();
        var $track = $(this);
        var $audio = $track.closest(".sns-audio-card").find(".sns-audio-native").first();
        var node = $audio.get(0);
        if (!node || !isFinite(node.duration) || node.duration <= 0) {
          return;
        }
        var rect = this.getBoundingClientRect();
        var ratio = Math.max(0, Math.min(1, (event.clientX - rect.left) / rect.width));
        node.currentTime = node.duration * ratio;
        updateAudioCard($audio);
      }).on("keydown.snsAudioPlayer", ".sns-audio-track", function(event) {
        if (event.key !== "ArrowLeft" && event.key !== "ArrowRight") {
          return;
        }
        event.preventDefault();
        var $audio = $(this).closest(".sns-audio-card").find(".sns-audio-native").first();
        var node = $audio.get(0);
        if (!node) {
          return;
        }
        node.currentTime = Math.max(0, Math.min(isFinite(node.duration) ? node.duration : Number.MAX_SAFE_INTEGER, node.currentTime + (event.key === "ArrowRight" ? 5 : -5)));
        updateAudioCard($audio);
      });
      $(document).off("click.snsVoiceMessage").on("click.snsVoiceMessage", ".sns-message .sns-voice-toggle", function(event) {
        event.preventDefault();
        event.stopPropagation();
        var $toggle = $(this);
        var $card = $toggle.closest(".sns-voice-card");
        var $post = $toggle.closest(".sns-message");
        var willOpen = !$card.hasClass("is-open");
        $card.toggleClass("is-open", willOpen);
        $post.toggleClass("sns-voice-expanded", willOpen);
        $toggle.attr("aria-expanded", willOpen ? "true" : "false");
      });
      $(document).off("click.snsReplyJump").on("click.snsReplyJump", ".sns-reply-preview", function(event) {
        event.preventDefault();
        event.stopPropagation();
        var targetId = String($(this).attr("data-sns-reply-target") || "").replace(/\D+/g, "");
        var $target = targetId ? $("#p" + targetId) : $();
        if (!$target.length) {
          return;
        }
        var topicNode = $("#sns-chat-shell>.topic").get(0);
        var targetNode = $target.get(0);
        if (topicNode && targetNode) {
          var topicRect = topicNode.getBoundingClientRect();
          var targetRect = targetNode.getBoundingClientRect();
          topicNode.scrollTo({
            top: topicNode.scrollTop + (targetRect.top - topicRect.top) - 28,
            behavior: "smooth"
          });
        }
        $target.removeClass("sns-reply-target-flash");
        void targetNode.offsetWidth;
        $target.addClass("sns-reply-target-flash");
        setTimeout(function() {
          $target.removeClass("sns-reply-target-flash");
        }, 1e3);
      });
      function scrollToBottom(smooth) {
        var element = $topic.get(0);
        if (!element) {
          return;
        }
        setTimeout(function() {
          try {
            element.scrollTo({
              top: element.scrollHeight,
              behavior: smooth ? "smooth" : "auto"
            });
          } catch (error) {
            element.scrollTop = element.scrollHeight;
          }
        }, 35);
      }
      $(document).off(".snsMessageMenu").on("click.snsMessageMenu", ".sns-menu-toggle", function(event) {
        event.preventDefault();
        event.stopPropagation();
        var $menu = $(this).siblings(".sns-menu");
        $(".sns-menu").not($menu).removeClass("is-open");
        $(".sns-message").not($menu.closest(".sns-message")).removeClass("sns-menu-active");
        var willOpen = !$menu.hasClass("is-open");
        $menu.toggleClass("is-open", willOpen);
        $menu.closest(".sns-message").toggleClass("sns-menu-active", willOpen);
      }).on("click.snsMessageMenu", function(event) {
        if (!$(event.target).closest(".sns-controls").length) {
          $(".sns-menu").removeClass("is-open");
          $(".sns-message").removeClass("sns-menu-active");
        }
      }).on("click.snsMessageMenu", ".sns-action-reply", function(event) {
        event.preventDefault();
        event.stopPropagation();
        var $post = $(this).closest(".post");
        $(".sns-menu").removeClass("is-open");
        $(".sns-message").removeClass("sns-menu-active");
        setReplyDraftFromPost($post);
      }).on("click.snsMessageMenu", ".sns-action-edit", function(event) {
        event.preventDefault();
        event.stopPropagation();
        var $post = $(this).closest(".post");
        var $link = findActionLink($post, "edit");
        $(".sns-menu").removeClass("is-open");
        $(".sns-message").removeClass("sns-menu-active");
        if ($link.length) {
          $link.get(0).click();
        }
      }).on("click.snsMessageMenu", ".sns-action-delete", function(event) {
        event.preventDefault();
        event.stopPropagation();
        var $post = $(this).closest(".post");
        var $link = findActionLink($post, "delete");
        $(".sns-menu").removeClass("is-open");
        $(".sns-message").removeClass("sns-menu-active");
        if ($link.length) {
          $link.get(0).click();
        }
      });
      function discardOldNativeAjaxSnapshot() {
        var baseline = parseInt(window.__SNS_LAST_CONFIRMED_TOPIC_BASELINE__, 10) || 0;
        if (!baseline) {
          return 0;
        }
        var removed = 0;
        $topic.find(".post.new-ajax[id]").each(function() {
          var $post = $(this);
          var id = parseInt(String($post.attr("id") || "").replace(/^p/i, "").replace(/\D+/g, ""), 10) || 0;
          if (id && id <= baseline) {
            $post.remove();
            removed++;
          }
        });
        return removed;
      }
      $(document).on("pun_post.sns", function() {
        var $input = $("#sns-ui-input");
        if (typeof window.SNSConfirmQueuedSend === "function") {
          window.SNSConfirmQueuedSend();
        }
        discardOldNativeAjaxSnapshot();
        [ 0, 40, 120, 300, 700 ].forEach(function(delay) {
          setTimeout(function() {
            if (typeof window.SNSHidePublishedNotice === "function") {
              window.SNSHidePublishedNotice();
            } else {
              hidePublishedRusffNotice();
            }
          }, delay);
        });
        enhancePosts();
        hideNativeForm();
        setTimeout(function() {
          discardOldNativeAjaxSnapshot();
          if (typeof window.SNSNormalizeNativeAjaxPosts === "function") {
            window.SNSNormalizeNativeAjaxPosts();
          }
          enhancePosts();
          hideNativeForm();
          scrollToBottom(true);
          $input.focus();
        }, 20);
        setTimeout(function() {
          window.__SNS_LAST_CONFIRMED_TOPIC_BASELINE__ = 0;
        }, 2500);
      });
      var snsEnhanceTimer = null;
      var snsPreserveScroll = false;
      if (window.MutationObserver) {
        var observer = new MutationObserver(function(mutations) {
          var dirty = mutations.some(function(mutation) {
            return Array.prototype.some.call(mutation.addedNodes, function(node) {
              return node.nodeType === 1 && node.matches(".post:not(.sns-tech-post):not(.sns-config-post)") && (!node.__snsEnhancedContent || node.__snsEnhancedContent !== node.querySelector(".post-content"));
            });
          });
          if (!dirty) return;
          snsPreserveScroll = snsPreserveScroll || window.__SNS_LAZY_HISTORY_PREPENDING__ === true;
          if (snsEnhanceTimer) return;
          snsEnhanceTimer = setTimeout(function() {
            snsEnhanceTimer = null;
            var preserve = snsPreserveScroll;
            snsPreserveScroll = false;
            discardOldNativeAjaxSnapshot();
            enhancePosts();
            if (!preserve) scrollToBottom(true);
          }, 40);
        });
        observer.observe($topic.get(0), {
          childList: true,
          subtree: false
        });
      }
      var snsEditingPost = null;
      document.addEventListener("click", function(event) {
        var edit = event.target.closest && event.target.closest('.sns-action-edit, a[href*="edit.php?id="]');
        if (edit) snsEditingPost = edit.closest(".post");
      }, true);
      $(document).on("pun_edit.snsPerf", function() {
        var post = snsEditingPost;
        snsEditingPost = null;
        setTimeout(function() {
          enhancePosts(post || true);
        }, 60);
      });
      createHeader();
      createShell();
      enhancePosts();
      createComposer();
      window.SNSEnhancePosts = function(post) {
        enhancePosts(post || true);
      };
      window.SNSNormalizeNativeAjaxPosts = normalizeNativeAjaxPosts;
    })(jQuery);
    (function($) {
      "use strict";
      var APP_ID = 16777213;
      var KEY_PREFIX = "snsrx2_t";
      var PALETTE = [ "\u2764\ufe0f", "\ud83d\udc94", "\ud83d\ude02", "\ud83d\udd25", "\ud83d\ude2d", "\ud83d\ude2e", "\ud83d\udc40", "\ud83d\udc4d", "\ud83d\udc80", "\ud83d\ude0f", "\ud83d\ude48", "\u2764\ufe0f\u200d\ud83d\udd25", "\ud83d\udc8b", "\ud83d\ude0d", "\ud83e\udee6", "\ud83d\ude49", "\ud83d\ude4a", "\ud83d\ude0e", "\ud83e\udd2c", "\ud83d\ude08", "\u2620\ufe0f", "\ud83d\udc7b", "\ud83d\udca9", "\ud83e\udd21", "\ud83e\udd1d", "\ud83d\ude4f", "\ud83e\udd7a", "\ud83c\udf1a", "\ud83d\udcaf", "\ud83d\uddff" ];
      var EMOJI_TO_CODE = {
        "\u2764\ufe0f": "h",
        "\ud83d\udc94": "b",
        "\ud83d\ude02": "j",
        "\ud83d\udd25": "f",
        "\ud83d\ude2d": "c",
        "\ud83d\ude2e": "o",
        "\ud83d\udc40": "e",
        "\ud83d\udc4d": "u",
        "\ud83d\udc80": "s",
        "\ud83d\ude0f": "A",
        "\ud83d\ude48": "B",
        "\u2764\ufe0f\u200d\ud83d\udd25": "C",
        "\ud83d\udc8b": "D",
        "\ud83d\ude0d": "E",
        "\ud83e\udee6": "F",
        "\ud83d\ude49": "G",
        "\ud83d\ude4a": "H",
        "\ud83d\ude0e": "I",
        "\ud83e\udd2c": "J",
        "\ud83d\ude08": "K",
        "\u2620\ufe0f": "L",
        "\ud83d\udc7b": "M",
        "\ud83d\udca9": "N",
        "\ud83e\udd21": "O",
        "\ud83e\udd1d": "P",
        "\ud83d\ude4f": "Q",
        "\ud83e\udd7a": "R",
        "\ud83c\udf1a": "S",
        "\ud83d\udcaf": "T",
        "\ud83d\uddff": "U"
      };
      var CODE_TO_EMOJI = {
        h: "\u2764\ufe0f",
        b: "\ud83d\udc94",
        j: "\ud83d\ude02",
        f: "\ud83d\udd25",
        c: "\ud83d\ude2d",
        o: "\ud83d\ude2e",
        e: "\ud83d\udc40",
        u: "\ud83d\udc4d",
        s: "\ud83d\udc80",
        A: "\ud83d\ude0f",
        B: "\ud83d\ude48",
        C: "\u2764\ufe0f\u200d\ud83d\udd25",
        D: "\ud83d\udc8b",
        E: "\ud83d\ude0d",
        F: "\ud83e\udee6",
        G: "\ud83d\ude49",
        H: "\ud83d\ude4a",
        I: "\ud83d\ude0e",
        J: "\ud83e\udd2c",
        K: "\ud83d\ude08",
        L: "\u2620\ufe0f",
        M: "\ud83d\udc7b",
        N: "\ud83d\udca9",
        O: "\ud83e\udd21",
        P: "\ud83e\udd1d",
        Q: "\ud83d\ude4f",
        R: "\ud83e\udd7a",
        S: "\ud83c\udf1a",
        T: "\ud83d\udcaf",
        U: "\ud83d\uddff"
      };
      var STYLE_ID = "sns-custom-reactions-style";
      var currentTopic = "";
      var currentUser = "";
      var currentUserName = "";
      var stateByUser = {};
      var saveBusy = false;
      var loadTimer = null;
      var reactionReadBusy = false;
      var lastReactionReadAt = 0;
      var pollTimer = null;
      var observer = null;
      var $sharedPicker = null;
      var pickerOwner = null;
      var pickerControl = null;
      var pickerObserver = null;
      function clean(value) {
        return String(value || "").replace(/\s+/g, " ").trim();
      }
      function topicId() {
        try {
          return String(new URL(location.href).searchParams.get("id") || "").replace(/\D+/g, "");
        } catch (error) {
          return "";
        }
      }
      function userIdFromHref(href) {
        var match = String(href || "").match(/profile\.php\?id=(\d+)/i);
        return match ? String(match[1]) : "";
      }
      function detectCurrentUserId() {
        var globals = [ window.UserID, window.UserId, window.USER_ID, window.user_id ];
        for (var i = 0; i < globals.length; i++) {
          if (globals[i] !== undefined && globals[i] !== null && /^\d+$/.test(String(globals[i])) && String(globals[i]) !== "0") {
            return String(globals[i]);
          }
        }
        var name = clean(typeof window.UserLogin !== "undefined" ? window.UserLogin : "");
        var result = "";
        $('#pun-status a[href*="profile.php?id="],' + '#pun-navlinks a[href*="profile.php?id="],' + ".sns-head-participants [data-participant-id]").each(function() {
          var $node = $(this);
          var id = String($node.attr("data-participant-id") || userIdFromHref($node.attr("href")) || "").replace(/\D+/g, "");
          if (!id) {
            return;
          }
          var nodeName = clean($node.attr("data-participant-name") || $node.text());
          if (!name || nodeName === name) {
            result = id;
            if (name && nodeName === name) {
              return false;
            }
          }
        });
        return result;
      }
      function detectCurrentUserName() {
        if (typeof window.UserLogin !== "undefined" && window.UserLogin) {
          return clean(window.UserLogin);
        }
        return clean($("#pun-status .item1, #pun-status .status_user").first().text());
      }
      function canReact() {
        return !!currentUser && !document.body.classList.contains("sns-chat-readonly");
      }
      function participantMap() {
        var map = {};
        $(".sns-head-participants [data-participant-id]").each(function() {
          var $node = $(this);
          var id = String($node.attr("data-participant-id") || "").replace(/\D+/g, "");
          if (!id) {
            return;
          }
          map[id] = clean($node.attr("data-participant-name") || $node.attr("title") || (id === currentUser ? currentUserName : "#" + id));
        });
        $("#pun-viewtopic .sns-message").each(function() {
          var $post = $(this);
          var id = String($post.attr("data-user-id") || $post.find("[data-user-id]").first().attr("data-user-id") || userIdFromHref($post.find('a[href*="profile.php?id="]').first().attr("href")) || "").replace(/\D+/g, "");
          if (!id || map[id]) {
            return;
          }
          map[id] = clean($post.find(".sns-author-link, .pa-author").first().text()) || (id === currentUser ? currentUserName : "#" + id);
        });
        if (currentUser && !map[currentUser]) {
          map[currentUser] = currentUserName || "#" + currentUser;
        }
        return map;
      }
      function participantIds() {
        return Object.keys(participantMap()).filter(function(id) {
          return /^\d+$/.test(id) && id !== "0";
        });
      }
      function postId($post) {
        return String($post.attr("id") || "").replace(/^p/i, "").replace(/\D+/g, "");
      }
      function safeParse(value) {
        if (!value) {
          return {};
        }
        if (typeof value === "object" && !Array.isArray(value)) {
          return value;
        }
        try {
          var parsed = JSON.parse(String(value));
          return parsed && typeof parsed === "object" && !Array.isArray(parsed) ? parsed : {};
        } catch (error) {
          return {};
        }
      }
      function normalizeReactionList(value) {
        var list = [];
        if (Array.isArray(value)) {
          list = value.slice();
        } else if (typeof value === "string" && value) {
          list = [ value ];
        }
        var seen = {};
        var result = [];
        PALETTE.forEach(function(emoji) {
          var hasEmoji = list.some(function(item) {
            return String(item || "") === emoji;
          });
          if (hasEmoji && !seen[emoji] && result.length < 3) {
            seen[emoji] = true;
            result.push(emoji);
          }
        });
        return result;
      }
      function sanitizeUserState(value) {
        var source = value && typeof value === "object" ? value : {};
        var result = {};
        Object.keys(source).forEach(function(pid) {
          var cleanPostId = String(pid).replace(/\D+/g, "");
          var list = normalizeReactionList(source[pid]);
          if (cleanPostId && list.length) {
            result[cleanPostId] = list;
          }
        });
        return result;
      }
      function statesEqual(a, b) {
        return JSON.stringify(sanitizeUserState(a || {})) === JSON.stringify(sanitizeUserState(b || {}));
      }
      function allReactionStatesEqual(a, b) {
        a = a && typeof a === "object" ? a : {};
        b = b && typeof b === "object" ? b : {};
        var aKeys = Object.keys(a).sort();
        var bKeys = Object.keys(b).sort();
        if (aKeys.length !== bKeys.length) {
          return false;
        }
        for (var i = 0; i < aKeys.length; i++) {
          if (aKeys[i] !== bKeys[i] || !statesEqual(a[aKeys[i]], b[bKeys[i]])) {
            return false;
          }
        }
        return true;
      }
      function ticket() {
        if (typeof window.ForumAPITicket !== "undefined" && window.ForumAPITicket) {
          return window.ForumAPITicket;
        }
        try {
          if (typeof ForumAPITicket !== "undefined" && ForumAPITicket) {
            return ForumAPITicket;
          }
        } catch (error) {}
        return "";
      }
      function storageKey(userId) {
        return KEY_PREFIX + currentTopic + "_u" + String(userId || "").replace(/\D+/g, "");
      }
      function encodeState(state) {
        state = sanitizeUserState(state || {});
        var ascii = {};
        Object.keys(state).forEach(function(pid) {
          var codes = (state[pid] || []).map(function(emoji) {
            return EMOJI_TO_CODE[emoji] || "";
          }).filter(Boolean).slice(0, 3);
          if (codes.length) {
            ascii[String(pid)] = codes.join("");
          }
        });
        return JSON.stringify(ascii);
      }
      function decodeState(raw) {
        var source = safeParse(raw);
        var result = {};
        Object.keys(source || {}).forEach(function(pid) {
          var cleanPostId = String(pid || "").replace(/\D+/g, "");
          var rawValue = source[pid];
          var emojis = [];
          if (Array.isArray(rawValue)) {
            emojis = rawValue.map(function(code) {
              var key = String(code || "");
              return CODE_TO_EMOJI[key] || CODE_TO_EMOJI[key.toLowerCase()] || "";
            });
          } else {
            var stringValue = String(rawValue || "");
            if (PALETTE.indexOf(stringValue) !== -1) {
              emojis = [ stringValue ];
            } else {
              emojis = stringValue.split("").map(function(code) {
                var key = String(code || "");
                return CODE_TO_EMOJI[key] || CODE_TO_EMOJI[key.toLowerCase()] || "";
              });
            }
          }
          emojis = normalizeReactionList(emojis);
          if (cleanPostId && emojis.length) {
            result[cleanPostId] = emojis;
          }
        });
        return result;
      }
      function installStyles() {
        var old = document.getElementById(STYLE_ID);
        if (old) {
          old.remove();
        }
        var style = document.createElement("style");
        style.id = STYLE_ID;
        style.textContent = [ "body.sns-chat-page #pun-viewtopic .reactions-root{display:none!important;}", "body.sns-chat-page .sns-reaction-chips{", "position:relative!important;left:auto!important;right:auto!important;bottom:auto!important;display:flex!important;align-items:center!important;flex-wrap:wrap!important;gap:4px!important;", "box-sizing:border-box!important;width:100%!important;max-width:100%!important;min-height:22px!important;margin:7px 0 0!important;padding:0!important;", "z-index:184!important;pointer-events:auto!important;}", "body.sns-chat-page .sns-own .post-content>.sns-reaction-chips{justify-content:flex-start!important;}", "body.sns-chat-page .sns-other .post-content>.sns-reaction-chips{justify-content:flex-end!important;}", "body.sns-chat-page .sns-reaction-chip{display:inline-flex!important;align-items:center!important;justify-content:center!important;gap:4px!important;", "box-sizing:border-box!important;height:22px!important;min-width:28px!important;margin:0!important;padding:1px 8px!important;", "border:1px solid rgba(127,127,127,.10)!important;border-radius:11px!important;background:rgba(127,127,127,.06)!important;color:inherit!important;", "background:color-mix(in srgb,currentColor 6%,transparent)!important;border-color:color-mix(in srgb,currentColor 11%,transparent)!important;", "box-shadow:none!important;backdrop-filter:blur(3px)!important;-webkit-backdrop-filter:blur(3px)!important;font:700 9px/1 Arial,sans-serif!important;", "cursor:pointer!important;pointer-events:auto!important;touch-action:manipulation!important;transition:transform .12s ease,background .12s ease,border-color .12s ease!important;}", "body.sns-chat-page .sns-reaction-chip:hover{transform:translateY(-1px)!important;background:rgba(127,127,127,.10)!important;background:color-mix(in srgb,currentColor 10%,transparent)!important;border-color:color-mix(in srgb,currentColor 16%,transparent)!important;}", "body.sns-chat-page .sns-reaction-chip.is-mine{background:rgba(127,127,127,.11)!important;background:color-mix(in srgb,currentColor 11%,transparent)!important;border-color:color-mix(in srgb,currentColor 18%,transparent)!important;}", 'body.sns-chat-page .sns-reaction-emoji{display:block!important;line-height:1!important;opacity:1!important;filter:none!important;font:14px/1 "Segoe UI Emoji","Apple Color Emoji","Noto Color Emoji",sans-serif!important;}', "body.sns-chat-page .sns-reaction-count{display:inline-block!important;min-width:6px!important;font:700 9px/1 Arial,sans-serif!important;opacity:1!important;color:inherit!important;text-shadow:none!important;text-align:center!important;}", "body.sns-chat-page .sns-reaction-control{position:absolute!important;bottom:-21px!important;display:block!important;width:23px!important;height:20px!important;", "margin:0!important;padding:0!important;z-index:190!important;pointer-events:auto!important;opacity:.74!important;", "transition:opacity .14s ease!important;}", "body.sns-chat-page .sns-own .sns-reaction-control{right:35px!important;left:auto!important;}", "body.sns-chat-page .sns-other .sns-reaction-control{left:35px!important;right:auto!important;}", "body.sns-chat-page .sns-reaction-add{display:flex!important;align-items:center!important;justify-content:center!important;box-sizing:border-box!important;", "width:23px!important;height:20px!important;margin:0!important;padding:0!important;border:0!important;background:transparent!important;", "color:rgba(255,255,255,.88)!important;filter:drop-shadow(0 1px 2px rgba(20,20,20,.58))!important;", "cursor:pointer!important;opacity:.92!important;pointer-events:auto!important;touch-action:manipulation!important;transition:color .12s ease,transform .12s ease!important;}", "body.sns-chat-page .sns-reaction-add:hover{color:#fff!important;transform:scale(1.10)!important;}", "body.sns-chat-page .sns-reaction-add:active{transform:scale(.94)!important;}", "body.sns-chat-page .sns-reaction-add svg{display:block!important;width:17px!important;height:17px!important;fill:none!important;stroke:currentColor!important;", "stroke-width:2.05!important;stroke-linecap:round!important;stroke-linejoin:round!important;pointer-events:none!important;}", "body.sns-chat-page #pun-viewtopic .sns-message .sns-menu-toggle{opacity:.62!important;pointer-events:auto!important;}", "body.sns-chat-page #pun-viewtopic .sns-message:hover .sns-menu-toggle,", "body.sns-chat-page #pun-viewtopic .sns-message.sns-menu-active .sns-menu-toggle,", "body.sns-chat-page #pun-viewtopic .sns-message:hover .sns-reaction-control,", "body.sns-chat-page #pun-viewtopic .sns-message.sns-menu-active .sns-reaction-control{opacity:1!important;pointer-events:auto!important;}", "body.sns-chat-page .sns-reaction-picker-custom{position:absolute!important;bottom:27px!important;display:none!important;", "grid-template-columns:repeat(6,30px)!important;grid-auto-rows:30px!important;align-items:center!important;justify-items:center!important;gap:3px!important;", "box-sizing:border-box!important;width:213px!important;max-width:88vw!important;padding:6px!important;", "border:1px solid rgba(255,255,255,.22)!important;border-radius:14px!important;", "background:rgba(29,26,23,.96)!important;box-shadow:0 10px 28px rgba(0,0,0,.28)!important;backdrop-filter:blur(10px)!important;", "-webkit-backdrop-filter:blur(10px)!important;white-space:normal!important;z-index:12050!important;pointer-events:auto!important;}", "body.sns-chat-page .sns-own .sns-reaction-picker-custom{right:0!important;left:auto!important;}", "body.sns-chat-page .sns-other .sns-reaction-picker-custom{left:0!important;right:auto!important;}", "body.sns-chat-page .sns-reaction-picker-custom.is-open{display:grid!important;}", "body.sns-chat-page .sns-reaction-choice{display:flex!important;align-items:center!important;justify-content:center!important;width:30px!important;height:30px!important;", "margin:0!important;padding:0!important;border:0!important;border-radius:9px!important;background:transparent!important;", 'font:18px/1 "Segoe UI Emoji","Apple Color Emoji","Noto Color Emoji",sans-serif!important;cursor:pointer!important;pointer-events:auto!important;', "touch-action:manipulation!important;transition:background .12s ease,transform .12s ease!important;}", "body.sns-chat-page .sns-reaction-choice:hover{background:rgba(255,255,255,.12)!important;transform:scale(1.14)!important;}", "body.sns-chat-page .sns-reaction-choice.is-current{background:rgba(255,255,255,.16)!important;}", "body.sns-chat-page .sns-reaction-error{display:none!important;position:absolute!important;bottom:27px!important;padding:5px 7px!important;", "border-radius:7px!important;background:rgba(32,28,25,.94)!important;color:#fff!important;font:9px/1.25 Arial,sans-serif!important;", "white-space:nowrap!important;box-shadow:0 6px 18px rgba(0,0,0,.22)!important;}", "body.sns-chat-page .sns-reaction-error.is-visible{display:block!important;}", "body.sns-chat-page .sns-own .sns-reaction-error{right:0!important;}", "body.sns-chat-page .sns-other .sns-reaction-error{left:0!important;}", "body.sns-chat-page .sns-message.sns-media-only .post-content>.sns-reaction-chips{width:calc(100% - 16px)!important;margin:7px 8px 8px!important;}", "body.sns-chat-page .sns-message.sns-media-caption .post-content>.sns-reaction-chips{margin-top:7px!important;}", "body.sns-chat-page .sns-message.sns-has-reaction-chips{margin-bottom:0!important;z-index:2!important;}", "body.sns-chat-page .sns-message.sns-has-reaction-chips:hover{z-index:35!important;}", "body.sns-chat-page.sns-chat-readonly .sns-reaction-control{display:none!important;}", "body.sns-chat-page.sns-chat-readonly .sns-reaction-chip{cursor:default!important;pointer-events:none!important;}", "@media(max-width:650px){", "body.sns-chat-page .sns-reaction-picker-custom{max-width:88vw!important;overflow:visible!important;}", "body.sns-chat-page .sns-own .sns-reaction-control{right:33px!important;}", "body.sns-chat-page .sns-other .sns-reaction-control{left:33px!important;}", "body.sns-chat-page #pun-viewtopic .sns-message .sns-menu-toggle{opacity:.58!important;pointer-events:auto!important;}", "body.sns-chat-page #pun-viewtopic .sns-message .sns-reaction-control{opacity:.72!important;pointer-events:auto!important;}", "}" ].join("");
        document.head.appendChild(style);
      }
      function removeNativeReactionUi() {
        $("#pun-viewtopic .reactions-root").remove();
        try {
          if (window.ReactionsPlugin && typeof window.ReactionsPlugin.setConfig === "function") {
            window.ReactionsPlugin.setConfig({
              disable: true
            });
          }
        } catch (error) {}
      }
      function aggregatedForPost(id, participants) {
        participants = participants || participantMap();
        var groups = {};
        Object.keys(stateByUser).forEach(function(userId) {
          var list = stateByUser[userId] && stateByUser[userId][id];
          list = normalizeReactionList(list);
          if (!list.length) {
            return;
          }
          list.forEach(function(emoji) {
            if (!groups[emoji]) {
              groups[emoji] = {
                emoji: emoji,
                users: []
              };
            }
            groups[emoji].users.push({
              id: userId,
              name: participants[userId] || (userId === currentUser ? currentUserName : "#" + userId)
            });
          });
        });
        return PALETTE.map(function(emoji) {
          return groups[emoji] || null;
        }).filter(Boolean);
      }
      function currentReactions(id) {
        return currentUser && stateByUser[currentUser] && stateByUser[currentUser][id] ? normalizeReactionList(stateByUser[currentUser][id]) : [];
      }
      function reactionAddSvg() {
        return '<svg viewBox="0 0 24 24" aria-hidden="true">' + '<circle cx="10" cy="10" r="6.5"></circle>' + '<path d="M7.5 11.5c.8 1.1 1.7 1.6 2.7 1.6s1.9-.5 2.7-1.6"></path>' + '<path d="M7.8 8.2h.01M12.2 8.2h.01"></path>' + '<path d="M18 13v6M15 16h6"></path>' + "</svg>";
      }
      function bindStopPropagation($node) {
        $node.on("pointerdown.snsCustomReactions mousedown.snsCustomReactions touchstart.snsCustomReactions", function(event) {
          event.stopPropagation();
        });
      }
      function closeReactionPicker() {
        if (pickerControl) {
          $(pickerControl).children(".sns-reaction-add").attr("aria-expanded", "false");
        }
        if ($sharedPicker) {
          $sharedPicker.removeClass("is-open").detach();
        }
        pickerOwner = null;
        pickerControl = null;
        if (pickerObserver) {
          pickerObserver.disconnect();
        }
      }
      function pickerOwnerIsCurrent() {
        return !!pickerOwner && !!pickerControl && canReact() && $.contains(document, pickerOwner) && $.contains(pickerOwner, pickerControl) && $sharedPicker && $sharedPicker.parent().get(0) === pickerControl;
      }
      function ensureSharedPicker() {
        if (!$sharedPicker) {
          $sharedPicker = $('<div class="sns-reaction-picker-custom"></div>');
          var pickerNode = $sharedPicker.get(0);
          // Native handlers survive jQuery cleanup when an external edit removes the owner.
          [ "pointerdown", "mousedown", "touchstart" ].forEach(function(type) {
            pickerNode.addEventListener(type, function(event) {
              event.stopPropagation();
            });
          });
          pickerNode.addEventListener("click", function(event) {
            event.preventDefault();
            event.stopPropagation();
            var $choice = $(event.target).closest(".sns-reaction-choice", pickerNode);
            if (!$choice.length || !pickerOwnerIsCurrent()) {
              return;
            }
            var $post = $(pickerOwner);
            var emoji = String($choice.attr("data-reaction") || "");
            closeReactionPicker();
            if (PALETTE.indexOf(emoji) !== -1) {
              setReaction($post, emoji);
            }
          });
        }
        if ($sharedPicker.children().length !== PALETTE.length) {
          $sharedPicker.empty();
          PALETTE.forEach(function(emoji) {
            $sharedPicker.append($('<button type="button" class="sns-reaction-choice"></button>').attr("data-reaction", emoji).text(emoji));
          });
        }
        return $sharedPicker;
      }
      function toggleReactionPicker($control) {
        var $post = $control.closest(".sns-message");
        if (!canReact() || !$post.length || !$.contains(document, $post.get(0))) {
          closeReactionPicker();
          return;
        }
        if (pickerControl === $control.get(0) && $sharedPicker && $sharedPicker.hasClass("is-open")) {
          closeReactionPicker();
          return;
        }
        closeReactionPicker();
        var $picker = ensureSharedPicker();
        var mine = currentReactions(postId($post));
        $picker.children(".sns-reaction-choice").each(function() {
          $(this).toggleClass("is-current", mine.indexOf(String($(this).attr("data-reaction") || "")) !== -1);
        });
        pickerOwner = $post.get(0);
        pickerControl = $control.get(0);
        // Keeping the picker inside this control preserves the own/other CSS alignment.
        $picker.appendTo($control).addClass("is-open");
        $control.children(".sns-reaction-add").attr("aria-expanded", "true");
        if (window.MutationObserver) {
          if (!pickerObserver) {
            pickerObserver = new MutationObserver(function() {
              if (!pickerOwnerIsCurrent()) {
                closeReactionPicker();
              }
            });
          }
          // Observe only while open so deleted or replaced owners do not retain the palette.
          pickerObserver.observe(document.body, { childList: true, subtree: true });
        }
      }
      function renderPost($post, participants) {
        if (!$post || !$post.length || !$post.hasClass("sns-message")) {
          return;
        }
        var id = postId($post);
        if (!id) {
          return;
        }
        var $body = $post.find(".post-body").first();
        if (!$body.length) {
          return;
        }
        var $content = $body.find(".post-content").first();
        if (!$content.length) {
          return;
        }
        $body.children(".sns-reaction-chips").remove();
        $body.children(".post-box").children(".sns-reaction-chips").remove();
        var groups = aggregatedForPost(id, participants);
        var mine = currentReactions(id);
        var canReactNow = canReact();
        var renderSignature = JSON.stringify([ groups, mine, canReactNow ]);
        var previousSignature = String($post.attr("data-sns-rx-render") || "");
        var existingChips = $content.children(".sns-reaction-chips").length > 0;
        var existingControl = $body.children(".sns-reaction-control").length > 0;
        if (previousSignature === renderSignature && (groups.length ? existingChips : !existingChips) && (canReactNow ? existingControl : !existingControl)) {
          return;
        }
        if (pickerOwner === $post.get(0)) {
          // Detach before .empty() can clean up the shared picker with the old control.
          closeReactionPicker();
        }
        var $chips = $content.children(".sns-reaction-chips").first();
        if (!$chips.length) {
          $chips = $('<div class="sns-reaction-chips"></div>');
          $content.append($chips);
        }
        $chips.empty();
        groups.forEach(function(group) {
          var count = group.users.length;
          var names = group.users.map(function(user) {
            return user.name;
          }).join(", ");
          var $chip = $('<button type="button" class="sns-reaction-chip"></button>').attr("data-reaction", group.emoji).attr("title", names).toggleClass("is-mine", mine.indexOf(group.emoji) !== -1).append($('<span class="sns-reaction-emoji"></span>').text(group.emoji));
          if (count > 1) {
            $chip.append($('<span class="sns-reaction-count"></span>').text(count));
          }
          bindStopPropagation($chip);
          $chip.on("click.snsCustomReactions", function(event) {
            event.preventDefault();
            event.stopPropagation();
            if (!canReact()) {
              return;
            }
            setReaction($(this).closest(".sns-message"), String($(this).attr("data-reaction") || ""));
          });
          $chips.append($chip);
        });
        if (!groups.length) {
          $chips.remove();
        }
        var $control = $body.children(".sns-reaction-control").first();
        if (canReactNow) {
          if (!$control.length) {
            $control = $('<div class="sns-reaction-control"></div>');
            $body.append($control);
          }
          $control.empty();
          var $add = $('<button type="button" class="sns-reaction-add" ' + 'title="\u0414\u043e\u0431\u0430\u0432\u0438\u0442\u044c \u0440\u0435\u0430\u043a\u0446\u0438\u044e" aria-label="\u0414\u043e\u0431\u0430\u0432\u0438\u0442\u044c \u0440\u0435\u0430\u043a\u0446\u0438\u044e"></button>').html(reactionAddSvg());
          var $error = $('<div class="sns-reaction-error"></div>');
          bindStopPropagation($add);
          $add.on("click.snsCustomReactions", function(event) {
            event.preventDefault();
            event.stopPropagation();
            toggleReactionPicker($(this).closest(".sns-reaction-control"));
          });
          $add.attr("aria-expanded", "false");
          $control.append($add, $error);
        } else if ($control.length) {
          $control.remove();
        }
        $post.toggleClass("sns-has-reaction-chips", groups.length > 0);
        $post.attr("data-sns-rx-render", renderSignature);
      }
      function renderAll() {
        if (pickerOwner && !pickerOwnerIsCurrent()) {
          closeReactionPicker();
        }
        removeNativeReactionUi();
        var participants = participantMap();
        $("#pun-viewtopic .sns-message").each(function() {
          renderPost($(this), participants);
        });
      }
      function renderFresh() {
        if (pickerOwner && !pickerOwnerIsCurrent()) {
          closeReactionPicker();
        }
        removeNativeReactionUi();
        var participants = participantMap();
        $("#pun-viewtopic .sns-message").filter(function() {
          return !$(this).attr("data-sns-rx-render");
        }).each(function() {
          renderPost($(this), participants);
        });
      }
      function storageData(json) {
        return json && json.response && json.response.storage && json.response.storage.data && typeof json.response.storage.data === "object" ? json.response.storage.data : {};
      }
      function storageGetKeys(keys) {
        keys = (keys || []).filter(Boolean);
        if (!keys.length) {
          return $.Deferred().resolve({
            response: {
              storage: {
                app_id: String(APP_ID),
                data: {}
              }
            }
          }).promise();
        }
        return SNSRequest({
          url: "/api.php",
          type: "GET",
          dataType: "json",
          cache: false,
          data: {
            method: "storage.get",
            app_id: APP_ID,
            key: keys,
            _sns_rx: Date.now()
          }
        });
      }
      function storageSetOwn(asciiValue) {
        var apiToken = ticket();
        if (!apiToken) {
          return $.Deferred().resolve({
            error: {
              code: "SNS_NO_FORUM_API_TICKET",
              message: "ForumAPITicket unavailable"
            }
          }).promise();
        }
        return SNSRequest({
          url: "/api.php",
          type: "GET",
          dataType: "json",
          cache: false,
          data: {
            method: "storage.set",
            app_id: APP_ID,
            key: storageKey(currentUser),
            value: asciiValue,
            token: apiToken,
            _sns_rx_write: Date.now()
          }
        });
      }
      function showRowError($post, message) {
        var $error = $post && $post.length ? $post.find(".sns-reaction-error").first() : $();
        if (!$error.length) {
          return;
        }
        $error.text(message).addClass("is-visible");
        setTimeout(function() {
          $error.removeClass("is-visible");
        }, 3200);
      }
      function loadSharedState() {
        if (!currentTopic || document.hidden || saveBusy || reactionReadBusy) {
          return;
        }
        var ids = participantIds();
        if (currentUser && ids.indexOf(currentUser) === -1) {
          ids.push(currentUser);
        }
        var keys = ids.map(storageKey);
        reactionReadBusy = true;
        lastReactionReadAt = Date.now();
        storageGetKeys(keys).done(function(json) {
          if (json && json.error) {
            return;
          }
          var data = storageData(json);
          var next = {};
          ids.forEach(function(uid) {
            var raw = data[storageKey(uid)];
            if (raw === undefined || raw === null || raw === "") {
              return;
            }
            var userState = decodeState(raw);
            if (Object.keys(userState).length) {
              next[uid] = userState;
            }
          });
          if (saveBusy && currentUser && stateByUser[currentUser]) {
            next[currentUser] = sanitizeUserState(stateByUser[currentUser]);
          }
          var changed = !allReactionStatesEqual(stateByUser, next);
          stateByUser = next;
          if (changed) {
            renderAll();
          }
        }).fail(function() {}).always(function() {
          reactionReadBusy = false;
        });
      }
      function verifyWrite($post, oldState, expectedState, attempt) {
        attempt = Number(attempt || 0);
        storageGetKeys([ storageKey(currentUser) ]).done(function(json) {
          var data = storageData(json);
          var serverState = decodeState(data[storageKey(currentUser)]);
          if (statesEqual(serverState, expectedState)) {
            if (Object.keys(serverState).length) {
              stateByUser[currentUser] = serverState;
            } else {
              delete stateByUser[currentUser];
            }
            saveBusy = false;
            renderAll();
            scheduleLoad(100);
            return;
          }
          if (attempt < 6) {
            setTimeout(function() {
              verifyWrite($post, oldState, expectedState, attempt + 1);
            }, [ 160, 300, 520, 850, 1300, 2100 ][attempt]);
            return;
          }
          if (Object.keys(oldState).length) {
            stateByUser[currentUser] = sanitizeUserState(oldState);
          } else {
            delete stateByUser[currentUser];
          }
          saveBusy = false;
          renderAll();
          showRowError($post, "RusFF \u043d\u0435 \u043f\u043e\u0434\u0442\u0432\u0435\u0440\u0434\u0438\u043b \u043e\u0431\u0449\u0443\u044e \u0440\u0435\u0430\u043a\u0446\u0438\u044e");
        }).fail(function() {
          if (attempt < 6) {
            setTimeout(function() {
              verifyWrite($post, oldState, expectedState, attempt + 1);
            }, 600);
            return;
          }
          if (Object.keys(oldState).length) {
            stateByUser[currentUser] = sanitizeUserState(oldState);
          } else {
            delete stateByUser[currentUser];
          }
          saveBusy = false;
          renderAll();
          showRowError($post, "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043f\u0440\u043e\u0432\u0435\u0440\u0438\u0442\u044c \u0440\u0435\u0430\u043a\u0446\u0438\u044e \u043d\u0430 \u0441\u0435\u0440\u0432\u0435\u0440\u0435");
        });
      }
      function saveCurrentState($post, oldState) {
        var expectedState = sanitizeUserState(stateByUser[currentUser] || {});
        var asciiValue = encodeState(expectedState);
        storageSetOwn(asciiValue).done(function(json) {
          if (json && json.error) {
            if (Object.keys(oldState).length) {
              stateByUser[currentUser] = sanitizeUserState(oldState);
            } else {
              delete stateByUser[currentUser];
            }
            saveBusy = false;
            renderAll();
            showRowError($post, "RusFF \u043e\u0442\u043a\u043b\u043e\u043d\u0438\u043b \u043e\u0431\u0449\u0443\u044e \u0440\u0435\u0430\u043a\u0446\u0438\u044e");
            return;
          }
          setTimeout(function() {
            verifyWrite($post, oldState, expectedState, 0);
          }, 120);
        }).fail(function() {
          if (Object.keys(oldState).length) {
            stateByUser[currentUser] = sanitizeUserState(oldState);
          } else {
            delete stateByUser[currentUser];
          }
          saveBusy = false;
          renderAll();
          showRowError($post, "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u0437\u0430\u043f\u0438\u0441\u0430\u0442\u044c \u043e\u0431\u0449\u0443\u044e \u0440\u0435\u0430\u043a\u0446\u0438\u044e");
        });
      }
      function setReaction($post, emoji) {
        if (saveBusy || !canReact() || !$post || !$post.length) {
          return;
        }
        var id = postId($post);
        if (!id) {
          return;
        }
        if (!stateByUser[currentUser]) {
          stateByUser[currentUser] = {};
        }
        var oldState = sanitizeUserState(stateByUser[currentUser] || {});
        var currentList = normalizeReactionList(stateByUser[currentUser][id]);
        if (currentList.indexOf(emoji) !== -1) {
          currentList = currentList.filter(function(item) {
            return item !== emoji;
          });
        } else {
          if (currentList.length >= 3) {
            showRowError($post, "\u041c\u043e\u0436\u043d\u043e \u043f\u043e\u0441\u0442\u0430\u0432\u0438\u0442\u044c \u043d\u0435 \u0431\u043e\u043b\u044c\u0448\u0435 3 \u0440\u0435\u0430\u043a\u0446\u0438\u0439 \u043d\u0430 \u043e\u0434\u043d\u043e \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435");
            return;
          }
          currentList.push(emoji);
          currentList = normalizeReactionList(currentList);
        }
        if (currentList.length) {
          stateByUser[currentUser][id] = currentList;
        } else {
          delete stateByUser[currentUser][id];
        }
        if (!Object.keys(stateByUser[currentUser]).length) {
          delete stateByUser[currentUser];
        }
        saveBusy = true;
        renderAll();
        saveCurrentState($post, oldState);
      }
      function scheduleLoad(delay) {
        if (loadTimer) clearTimeout(loadTimer);
        if (document.hidden || reactionReadBusy) {
          loadTimer = null;
          return;
        }
        var wait = Math.max(delay === undefined ? 100 : delay, 3e3 - (Date.now() - lastReactionReadAt));
        loadTimer = setTimeout(function() {
          loadTimer = null;
          loadSharedState();
        }, wait);
      }
      function startPolling() {
        if (pollTimer) {
          clearInterval(pollTimer);
          pollTimer = null;
        }
      }
      function installObserver() {
        if (observer || !window.MutationObserver) {
          return;
        }
        var $watchRoot = $("#sns-chat-shell > .topic").first();
        if (!$watchRoot.length) {
          return;
        }
        observer = new MutationObserver(function(mutations) {
          var relevant = false;
          mutations.some(function(mutation) {
            return Array.prototype.some.call(mutation.addedNodes || [], function(node) {
              if (!node || node.nodeType !== 1) {
                return false;
              }
              var $node = $(node);
              if ($node.is(".sns-reaction-chips, .sns-reaction-control, .sns-reaction-chip, .sns-reaction-add, .sns-reaction-picker-custom, .sns-reaction-choice, .sns-reaction-error") || $node.closest(".sns-reaction-chips, .sns-reaction-control").length) {
                return false;
              }
              if ($node.is(".post, .sns-message") || $node.find(".post, .sns-message").length) {
                relevant = true;
                return true;
              }
              return false;
            });
          });
          if (relevant) {
            removeNativeReactionUi();
            setTimeout(renderAll, 70);
            scheduleLoad(110);
          }
        });
        observer.observe($watchRoot.get(0), {
          childList: true,
          subtree: true
        });
      }
      $(document).off(".snsCustomReactions .snsCustomReactionsOutside").on("pun_post.snsCustomReactions sns_pages_loaded.snsCustomReactions", function() {
        setTimeout(renderFresh, 90);
        if (arguments && arguments[0] && arguments[0].type === "sns_pages_loaded") {
          scheduleLoad(180);
        }
      }).on("pun_edit.snsCustomReactions", function() {
        setTimeout(renderAll, 45);
      }).on("click.snsCustomReactionsOutside", function(event) {
        if (!$(event.target).closest(".sns-reaction-control, .sns-reaction-chips").length) {
          closeReactionPicker();
        }
      }).on("keydown.snsCustomReactions", function(event) {
        if (event.key === "Escape") {
          closeReactionPicker();
        }
      });
      $(window).off("focus.snsCustomReactions").on("focus.snsCustomReactions", function() {
        if (!saveBusy) {
          scheduleLoad(150);
        }
      });
      document.addEventListener("visibilitychange", function() {
        if (document.visibilityState === "visible" && !saveBusy) {
          scheduleLoad(60);
        }
      });
      function start() {
        if (!document.body.classList.contains("sns-chat-page")) {
          return;
        }
        currentTopic = topicId();
        currentUser = detectCurrentUserId();
        currentUserName = detectCurrentUserName();
        installStyles();
        removeNativeReactionUi();
        renderAll();
        scheduleLoad(80);
        startPolling();
      }
      start();
    })(jQuery);
    (function($) {
      "use strict";
      var CUSTOM_STYLE_ID = "sns-chat-v07-custom-style";
      var CONFIG_PREFIX = "SNSCFG:";
      var DEFAULTS = {
        title: "",
        backgroundUrl: "",
        overlayColor: "#714b35",
        overlayOpacity: 36,
        noiseOpacity: 0,
        ownGradient1: "#b65f3a",
        ownGradient2: "#d7a44a",
        gradientAngle: 135,
        otherBubble: "#eee7dd",
        dialogMode: "solid",
        dialogColor: "#fbfaf7",
        dialogGradient1: "#fffaf5",
        dialogGradient2: "#eee2d4",
        dialogGradientAngle: 135,
        dialogPhotoUrl: "",
        dialogPhotoTint: "#fffaf5",
        dialogPhotoTintOpacity: 12,
        dialogNoiseOpacity: 0,
        dialogBlur: 0,
        dialogPattern: "dots",
        dialogPatternUrl: "",
        dialogPatternBase: "#fbfaf7",
        dialogPatternColor: "#d8c6b3",
        dialogPatternSize: 26,
        chatAvatar: "",
        avatarGrayscale: 0,
        avatarNoiseOpacity: 0,
        avatarTint: "#8a5a3a",
        avatarTintOpacity: 0,
        avatarBlendMode: "normal",
        participantsConfigured: false,
        participantsVersion: 2,
        participants: [],
        participantColors: {},
        configSchema: 67,
        configSavedAt: 0
      };
      var currentConfig = null;
      var uploadCapture = null;
      function cleanText(text) {
        return String(text || "").replace(/\s+/g, " ").trim();
      }
      function clamp(value, min, max) {
        value = Number(value);
        if (!isFinite(value)) value = min;
        return Math.max(min, Math.min(max, value));
      }
      function safeColor(value, fallback) {
        value = String(value || "").trim();
        return /^#[0-9a-f]{6}$/i.test(value) ? value : fallback;
      }
      function readableTextColor(background) {
        var hex = safeColor(background, "#ffffff").replace("#", "");
        var r = parseInt(hex.substring(0, 2), 16);
        var g = parseInt(hex.substring(2, 4), 16);
        var b = parseInt(hex.substring(4, 6), 16);
        var yiq = (r * 299 + g * 587 + b * 114) / 1e3;
        return yiq < 150 ? "#f7f7f8" : "#333333";
      }
      function safeChoice(value, allowed, fallback) {
        value = String(value || "").trim().toLowerCase();
        return allowed.indexOf(value) !== -1 ? value : fallback;
      }
      function hexToRgba(hex, alpha) {
        hex = safeColor(hex, "#ffffff").replace("#", "");
        var r = parseInt(hex.substring(0, 2), 16);
        var g = parseInt(hex.substring(2, 4), 16);
        var b = parseInt(hex.substring(4, 6), 16);
        alpha = clamp(alpha, 0, 100) / 100;
        return "rgba(" + r + "," + g + "," + b + "," + alpha + ")";
      }
      function escapedCssUrl(url) {
        url = String(url || "").trim();
        if (!url) {
          return "";
        }
        return 'url("' + url.replace(/["\\]/g, "\\$&") + '")';
      }
      function dialogBackgroundCss(config) {
        var mode = safeChoice(config.dialogMode, [ "solid", "gradient", "photo", "pattern" ], DEFAULTS.dialogMode);
        var result = {
          color: safeColor(config.dialogColor, DEFAULTS.dialogColor),
          image: "none",
          size: "cover",
          repeat: "no-repeat",
          position: "center"
        };
        if (mode === "gradient") {
          result.color = safeColor(config.dialogGradient1, DEFAULTS.dialogGradient1);
          result.image = "linear-gradient(" + clamp(config.dialogGradientAngle, 0, 360) + "deg," + safeColor(config.dialogGradient1, DEFAULTS.dialogGradient1) + "," + safeColor(config.dialogGradient2, DEFAULTS.dialogGradient2) + ")";
        } else if (mode === "photo") {
          result.color = safeColor(config.dialogColor, DEFAULTS.dialogColor);
          var photo = escapedCssUrl(config.dialogPhotoUrl);
          var tint = hexToRgba(config.dialogPhotoTint, config.dialogPhotoTintOpacity);
          result.image = photo ? "linear-gradient(" + tint + "," + tint + ")," + photo : "none";
          result.size = "cover";
          result.repeat = "no-repeat";
          result.position = "center";
        } else if (mode === "pattern") {
          result.color = safeColor(config.dialogPatternBase, DEFAULTS.dialogPatternBase);
          var pattern = safeChoice(config.dialogPattern, [ "dots", "grid", "diagonal", "custom" ], DEFAULTS.dialogPattern);
          var patternColor = safeColor(config.dialogPatternColor, DEFAULTS.dialogPatternColor);
          var size = clamp(config.dialogPatternSize, 10, 160);
          if (pattern === "custom" && String(config.dialogPatternUrl || "").trim()) {
            result.image = escapedCssUrl(config.dialogPatternUrl);
            result.size = size + "px " + size + "px";
            result.repeat = "repeat";
            result.position = "center";
          } else if (pattern === "grid") {
            result.image = "linear-gradient(" + patternColor + " 1px,transparent 1px)," + "linear-gradient(90deg," + patternColor + " 1px,transparent 1px)";
            result.size = size + "px " + size + "px";
            result.repeat = "repeat";
          } else if (pattern === "diagonal") {
            result.image = "repeating-linear-gradient(135deg," + "transparent 0," + "transparent " + Math.max(6, Math.round(size * .48)) + "px," + patternColor + " " + Math.max(7, Math.round(size * .48) + 1) + "px," + patternColor + " " + Math.max(8, Math.round(size * .48) + 2) + "px)";
            result.size = "auto";
            result.repeat = "repeat";
          } else {
            var dot = Math.max(1, Math.round(size / 18));
            result.image = "radial-gradient(circle," + patternColor + " " + dot + "px," + "transparent " + (dot + 1) + "px)";
            result.size = size + "px " + size + "px";
            result.repeat = "repeat";
          }
        }
        return result;
      }
      function getTopic() {
        return $("#sns-chat-shell > .topic, #pun-viewtopic .topic").first();
      }
      function getFirstPost() {
        return getTopic().find(".post").first();
      }
      function getPostAuthor($post) {
        var name = cleanText($post.find(".pa-author a").first().text());
        if (!name) {
          name = cleanText($post.find(".pa-author").first().text().replace(/^\u0410\u0432\u0442\u043E\u0440:\s*/i, ""));
        }
        return name;
      }
      function getPostAvatar($post) {
        var selectors = [ ".pa-avatar img", ".post-author .pa-avatar img", "img.avatardemo", ".post-author img.avatar" ];
        for (var i = 0; i < selectors.length; i++) {
          var src = $post.find(selectors[i]).first().attr("src");
          if (src) {
            return src;
          }
        }
        var fallback = "";
        $post.find(".post-author img[src]").each(function() {
          var $img = $(this);
          var src = String($img.attr("src") || "");
          var cls = String($img.attr("class") || "");
          var alt = String($img.attr("alt") || "");
          if (!src || /flag|icon|online|offline|rank|smil|emoji|blank\.gif/i.test(src + " " + cls + " " + alt)) {
            return;
          }
          var node = this;
          if (node.naturalWidth >= 40 || node.width >= 40) {
            fallback = src;
            return false;
          }
          if (!fallback) {
            fallback = src;
          }
        });
        return fallback;
      }
      function getCurrentUser() {
        if (typeof window.UserLogin !== "undefined" && window.UserLogin) {
          return cleanText(window.UserLogin);
        }
        var text = cleanText($("#pun-status .item1, #pun-status .status_user").first().text());
        var match = text.match(/(?:\u043F\u0440\u0438\u0432\u0435\u0442|hello)[,\s]+([^!,]+)/i);
        return match ? cleanText(match[1]) : "";
      }
      function profileIdFromHref(href) {
        var match = String(href || "").match(/profile\.php\?id=(\d+)/i);
        return match ? String(match[1]) : "";
      }
      function getPostUserId($post) {
        var directId = String($post.attr("data-user-id") || $post.find("[data-user-id]").first().attr("data-user-id") || "");
        if (/^\d+$/.test(directId)) {
          return directId;
        }
        return profileIdFromHref($post.find('.pa-author a[href*="profile.php?id="],' + 'a[href*="profile.php?id="]').first().attr("href") || "");
      }
      function getCurrentUserId() {
        var globals = [ window.UserID, window.UserId, window.USER_ID, window.user_id ];
        for (var i = 0; i < globals.length; i++) {
          if (globals[i] !== undefined && globals[i] !== null && /^\d+$/.test(String(globals[i]))) {
            return String(globals[i]);
          }
        }
        var currentName = participantKey(getCurrentUser());
        var result = "";
        $('#pun-status a[href*="profile.php?id="],' + '#pun-navlinks a[href*="profile.php?id="],' + 'a[href*="profile.php?id="]').each(function() {
          var $link = $(this);
          if (currentName && participantKey($link.text()) === currentName) {
            result = profileIdFromHref($link.attr("href"));
            if (result) {
              return false;
            }
          }
        });
        return result;
      }
      function ownerParticipant() {
        var $first = getFirstPost();
        return {
          id: getPostUserId($first),
          name: cleanText(getPostAuthor($first))
        };
      }
      function originalTopicTitle() {
        var customTitle = $("#sns-chat-header .sns-head-title").attr("data-sns-original-title");
        if (customTitle) return customTitle;
        var shown = cleanText($("#sns-chat-header .sns-head-title").first().text());
        if (shown) return shown;
        var crumbs = cleanText($("#pun-crumbs1 .crumbs, #pun-crumbs2 .crumbs").first().text());
        if (crumbs) {
          var bits = crumbs.split("\xbb");
          return cleanText(bits[bits.length - 1]);
        }
        return "SNS CHAT";
      }
      function encodeConfig(config) {
        var json = JSON.stringify(config);
        try {
          return btoa(unescape(encodeURIComponent(json)));
        } catch (error) {
          return btoa(json);
        }
      }
      function decodeConfig(encoded) {
        try {
          var json = decodeURIComponent(escape(atob(encoded)));
          return JSON.parse(json);
        } catch (error) {
          try {
            return JSON.parse(atob(encoded));
          } catch (secondError) {
            return null;
          }
        }
      }
      function snsTopicId() {
        var match = String(location.search || "").match(/(?:\?|&)id=(\d+)/);
        return match ? match[1] : "";
      }
      function snsConfigCacheKey() {
        return "SNS_CHAT_CONFIG_" + (snsTopicId() || location.pathname);
      }
      function snsConfigCacheMetaKey() {
        return snsConfigCacheKey() + "_META";
      }
      function cachedConfigMeta() {
        try {
          var raw = localStorage.getItem(snsConfigCacheMetaKey());
          return raw ? JSON.parse(raw) : {};
        } catch (error) {
          return {};
        }
      }
      function cacheConfigEncoded(encoded, meta) {
        if (!encoded) {
          return;
        }
        try {
          localStorage.setItem(snsConfigCacheKey(), encoded);
          if (meta && typeof meta === "object") {
            localStorage.setItem(snsConfigCacheMetaKey(), JSON.stringify(meta));
          }
        } catch (error) {}
      }
      function cachedConfigObject() {
        try {
          var encoded = localStorage.getItem(snsConfigCacheKey());
          return encoded ? decodeConfig(encoded) : null;
        } catch (error) {
          return null;
        }
      }
      function installRemoteConfigShadow(encoded) {
        if (!encoded) {
          return;
        }
        getTopic().children(".sns-config-shadow").remove();
        var $shadow = $('<div class="post sns-config-post sns-config-shadow" aria-hidden="true">' + '<div class="post-content"></div>' + "</div>");
        $shadow.find(".post-content").text(CONFIG_PREFIX + encoded);
        $shadow.css("display", "none");
        getTopic().append($shadow);
      }
      function lastTopicPageUrl() {
        var topicId = snsTopicId();
        if (!topicId) {
          return location.href;
        }
        var bestPage = 1;
        var bestUrl = location.href;
        $("a[href]").each(function() {
          var href = this.href || $(this).attr("href") || "";
          if (!href || !/viewtopic\.php/i.test(href)) {
            return;
          }
          var idMatch = href.match(/(?:\?|&)id=(\d+)/);
          if (!idMatch || idMatch[1] !== topicId) {
            return;
          }
          var pageMatch = href.match(/(?:\?|&)p=(\d+)/);
          var pageNumber = pageMatch ? parseInt(pageMatch[1], 10) : 1;
          if (isFinite(pageNumber) && pageNumber >= bestPage) {
            bestPage = pageNumber;
            bestUrl = href;
          }
        });
        return bestUrl;
      }
      function loadLatestConfigFromServer(callback) {
        var topicId = snsTopicId();
        if (!topicId) {
          if (callback) {
            callback(null);
          }
          return;
        }
        var apiUrl = new URL("api.php", location.href);
        apiUrl.searchParams.set("method", "post.get");
        apiUrl.searchParams.set("topic_id", topicId);
        apiUrl.searchParams.set("sort_by", "id");
        apiUrl.searchParams.set("sort_dir", "desc");
        apiUrl.searchParams.set("limit", "100");
        apiUrl.searchParams.set("fields", "id,message,topic_id,user_id");
        apiUrl.searchParams.set("_sns_config_load", String(Date.now()));
        SNSRequest({
          url: apiUrl.toString(),
          type: "GET",
          dataType: "json",
          cache: false,
          timeout: 5500
        }).done(function(json) {
          var posts = json && Array.isArray(json.response) ? json.response.slice() : [];
          if (!posts.length) {
            if (callback) {
              callback(null);
            }
            return;
          }
          posts.sort(function(a, b) {
            return parseInt(b.id, 10) - parseInt(a.id, 10);
          });
          var newestEncoded = "";
          var newestConfig = null;
          var newestPostId = 0;
          var decodedConfigHistory = [];
          for (var i = 0; i < posts.length; i++) {
            var post = posts[i] || {};
            var holder = document.createElement("div");
            holder.innerHTML = String(post.message || "");
            var raw = String(holder.textContent || holder.innerText || "");
            var match = raw.match(/SNSCFG:([A-Za-z0-9+\/=]+)/);
            if (!match) {
              continue;
            }
            var decoded = decodeConfig(match[1]);
            if (!decoded) {
              continue;
            }
            decodedConfigHistory.push({
              encoded: match[1],
              config: decoded,
              postId: parseInt(post.id, 10) || 0
            });
            if (!newestConfig) {
              newestEncoded = match[1];
              newestConfig = decoded;
              newestPostId = parseInt(post.id, 10) || 0;
            }
          }
          if (!newestEncoded || !newestConfig) {
            if (callback) {
              callback(null);
            }
            return;
          }
          var localConfig = cachedConfigObject();
          newestConfig = recoverLegacyAvatarStyle(newestConfig, [ localConfig ].concat(decodedConfigHistory.map(function(item) {
            return item.config;
          })));
          newestEncoded = encodeConfig(newestConfig);
          var localSavedAt = configSavedAtValue(localConfig);
          var remoteSavedAt = configSavedAtValue(newestConfig);
          var chosenConfig = newestConfig;
          var chosenEncoded = newestEncoded;
          if (localConfig) {
            if (localSavedAt && remoteSavedAt && localSavedAt > remoteSavedAt) {
              chosenConfig = localConfig;
              chosenEncoded = encodeConfig(localConfig);
            } else if (localSavedAt && !remoteSavedAt) {
              chosenConfig = localConfig;
              chosenEncoded = encodeConfig(localConfig);
            } else if (!localSavedAt && !remoteSavedAt) {
              var localScore = configCompletenessScore(localConfig);
              var remoteScore = configCompletenessScore(newestConfig);
              if (localScore >= remoteScore) {
                chosenConfig = localConfig;
                chosenEncoded = encodeConfig(localConfig);
              }
            }
          }
          chosenConfig = recoverLegacyAvatarStyle(chosenConfig, [ localConfig, newestConfig ].concat(decodedConfigHistory.map(function(item) {
            return item.config;
          })));
          chosenEncoded = encodeConfig(chosenConfig);
          cacheConfigEncoded(chosenEncoded, {
            source: chosenConfig === newestConfig ? "server" : "local-migration",
            serverPostId: newestPostId,
            savedAt: configSavedAtValue(chosenConfig),
            cachedAt: Date.now()
          });
          installRemoteConfigShadow(chosenEncoded);
          if (callback) {
            callback(readConfig());
          }
        }).fail(function() {
          if (callback) {
            callback(null);
          }
        });
      }
      function configSavedAtValue(config) {
        var value = Number(config && config.configSavedAt ? config.configSavedAt : 0);
        return isFinite(value) ? value : 0;
      }
      function avatarStyleIsCustomized(config) {
        config = config || {};
        return Number(config.avatarGrayscale || 0) > 0 || Number(config.avatarNoiseOpacity || 0) > 0 || Number(config.avatarTintOpacity || 0) > 0 || String(config.avatarBlendMode || "normal") !== "normal";
      }
      function sameChatAvatar(a, b) {
        var left = String(a && a.chatAvatar ? a.chatAvatar : "").trim();
        var right = String(b && b.chatAvatar ? b.chatAvatar : "").trim();
        if (left && right) {
          return left === right;
        }
        return !left && !right;
      }
      function recoverLegacyAvatarStyle(primaryConfig, candidates) {
        if (!primaryConfig || typeof primaryConfig !== "object") {
          return primaryConfig;
        }
        var result = $.extend({}, primaryConfig);
        if (Number(result.configSavedAt || 0) > 0 && Number(result.configSchema || 0) >= 67) {
          return result;
        }
        if (avatarStyleIsCustomized(result)) {
          return result;
        }
        candidates = Array.isArray(candidates) ? candidates : [];
        for (var i = 0; i < candidates.length; i++) {
          var candidate = candidates[i];
          if (!candidate || typeof candidate !== "object" || !sameChatAvatar(result, candidate) || !avatarStyleIsCustomized(candidate)) {
            continue;
          }
          result.avatarGrayscale = candidate.avatarGrayscale;
          result.avatarNoiseOpacity = candidate.avatarNoiseOpacity;
          result.avatarTint = candidate.avatarTint;
          result.avatarTintOpacity = candidate.avatarTintOpacity;
          result.avatarBlendMode = candidate.avatarBlendMode;
          result.__snsRecoveredAvatarStyle = true;
          break;
        }
        return result;
      }
      function configCompletenessScore(config) {
        if (!config || typeof config !== "object") {
          return 0;
        }
        var score = 0;
        if (config.participantColors && typeof config.participantColors === "object" && Object.keys(config.participantColors).length) {
          score += 6;
        }
        if (avatarStyleIsCustomized(config)) {
          score += 8;
        }
        if (String(config.chatAvatar || "").trim()) {
          score += 2;
        }
        if (config.dialogMode && config.dialogMode !== DEFAULTS.dialogMode) {
          score += 3;
        }
        if (String(config.dialogPhotoUrl || "").trim()) {
          score += 7;
        }
        if (String(config.dialogPatternUrl || "").trim()) {
          score += 7;
        }
        [ "dialogColor", "dialogGradient1", "dialogGradient2", "dialogPhotoTint", "dialogPatternBase", "dialogPatternColor" ].forEach(function(key) {
          if (config[key] !== undefined && String(config[key]) !== String(DEFAULTS[key])) {
            score++;
          }
        });
        if (String(config.backgroundUrl || "").trim()) {
          score++;
        }
        if (String(config.title || "").trim()) {
          score++;
        }
        return score;
      }
      function choosePreferredConfig(localConfig, domConfig) {
        if (!localConfig) {
          return domConfig;
        }
        if (!domConfig) {
          return localConfig;
        }
        var localSavedAt = configSavedAtValue(localConfig);
        var domSavedAt = configSavedAtValue(domConfig);
        if (localSavedAt && domSavedAt) {
          return localSavedAt >= domSavedAt ? localConfig : domConfig;
        }
        if (localSavedAt && !domSavedAt) {
          return localConfig;
        }
        if (domSavedAt && !localSavedAt) {
          return domConfig;
        }
        var localScore = configCompletenessScore(localConfig);
        var domScore = configCompletenessScore(domConfig);
        if (domScore > localScore + 4) {
          return domConfig;
        }
        return localConfig;
      }
      function readConfig() {
        var domSaved = null;
        getTopic().find(".post").each(function() {
          var raw = cleanText($(this).find(".post-content").first().text());
          var match = raw.match(/SNSCFG:([A-Za-z0-9+\/=]+)/);
          if (!match) {
            return;
          }
          var decoded = decodeConfig(match[1]);
          if (decoded) {
            domSaved = decoded;
          }
        });
        var localSaved = cachedConfigObject();
        var saved = choosePreferredConfig(localSaved, domSaved);
        saved = recoverLegacyAvatarStyle(saved, [ localSaved, domSaved ]);
        var merged = $.extend({}, DEFAULTS, saved || {});
        if (saved) {
          if (String(saved.overlayColor || "").toLowerCase() === "#6f94e8") merged.overlayColor = DEFAULTS.overlayColor;
          if (String(saved.ownGradient1 || "").toLowerCase() === "#7594ff") merged.ownGradient1 = DEFAULTS.ownGradient1;
          if ([ "#7ecfff", "#00ff00" ].indexOf(String(saved.ownGradient2 || "").toLowerCase()) !== -1) merged.ownGradient2 = DEFAULTS.ownGradient2;
          if (String(saved.otherBubble || "").toLowerCase() === "#ececf1") merged.otherBubble = DEFAULTS.otherBubble;
        }
        merged.overlayColor = safeColor(merged.overlayColor, DEFAULTS.overlayColor);
        merged.ownGradient1 = safeColor(merged.ownGradient1, DEFAULTS.ownGradient1);
        merged.ownGradient2 = safeColor(merged.ownGradient2, DEFAULTS.ownGradient2);
        merged.otherBubble = safeColor(merged.otherBubble, DEFAULTS.otherBubble);
        merged.dialogMode = safeChoice(merged.dialogMode, [ "solid", "gradient", "photo", "pattern" ], DEFAULTS.dialogMode);
        merged.dialogColor = safeColor(merged.dialogColor, DEFAULTS.dialogColor);
        merged.dialogGradient1 = safeColor(merged.dialogGradient1, DEFAULTS.dialogGradient1);
        merged.dialogGradient2 = safeColor(merged.dialogGradient2, DEFAULTS.dialogGradient2);
        merged.dialogPhotoTint = safeColor(merged.dialogPhotoTint, DEFAULTS.dialogPhotoTint);
        merged.dialogPatternBase = safeColor(merged.dialogPatternBase, DEFAULTS.dialogPatternBase);
        merged.dialogPatternColor = safeColor(merged.dialogPatternColor, DEFAULTS.dialogPatternColor);
        merged.avatarTint = safeColor(merged.avatarTint, DEFAULTS.avatarTint);
        merged.avatarBlendMode = safeChoice(merged.avatarBlendMode, [ "normal", "multiply", "soft-light", "color" ], DEFAULTS.avatarBlendMode);
        merged.dialogPattern = safeChoice(merged.dialogPattern, [ "dots", "grid", "diagonal", "custom" ], DEFAULTS.dialogPattern);
        merged.overlayOpacity = clamp(merged.overlayOpacity, 0, 90);
        merged.noiseOpacity = clamp(merged.noiseOpacity, 0, 100);
        merged.gradientAngle = clamp(merged.gradientAngle, 0, 360);
        merged.dialogGradientAngle = clamp(merged.dialogGradientAngle, 0, 360);
        merged.dialogPhotoTintOpacity = clamp(merged.dialogPhotoTintOpacity, 0, 80);
        merged.dialogNoiseOpacity = clamp(merged.dialogNoiseOpacity, 0, 100);
        merged.dialogBlur = clamp(merged.dialogBlur, 0, 18);
        merged.dialogPatternSize = clamp(merged.dialogPatternSize, 10, 160);
        merged.avatarGrayscale = clamp(merged.avatarGrayscale, 0, 100);
        merged.avatarNoiseOpacity = clamp(merged.avatarNoiseOpacity, 0, 100);
        merged.avatarTintOpacity = clamp(merged.avatarTintOpacity, 0, 80);
        merged.configSchema = Math.max(0, parseInt(merged.configSchema, 10) || 0);
        merged.configSavedAt = Math.max(0, Number(merged.configSavedAt) || 0);
        merged.title = String(merged.title || "").trim();
        merged.backgroundUrl = String(merged.backgroundUrl || "").trim();
        merged.dialogPhotoUrl = String(merged.dialogPhotoUrl || "").trim();
        merged.dialogPatternUrl = String(merged.dialogPatternUrl || "").trim();
        merged.chatAvatar = String(merged.chatAvatar || "").trim();
        merged.participantsConfigured = !!merged.participantsConfigured;
        merged.participantsVersion = 2;
        if (!Array.isArray(merged.participants)) {
          merged.participants = [];
        }
        var participantSeen = {};
        merged.participants = merged.participants.map(function(item) {
          if (typeof item === "string") {
            return {
              id: "",
              name: cleanText(item)
            };
          }
          item = item && typeof item === "object" ? item : {};
          var id = /^\d+$/.test(String(item.id || "")) ? String(item.id) : "";
          var name = cleanText(item.name || item.username || "");
          if (!name && !id) {
            return null;
          }
          return {
            id: id,
            name: name
          };
        }).filter(function(item) {
          if (!item) {
            return false;
          }
          var key = item.id ? "id:" + item.id : "name:" + participantKey(item.name);
          if (!key || participantSeen[key]) {
            return false;
          }
          participantSeen[key] = true;
          return true;
        });
        var rawParticipantColors = merged.participantColors && typeof merged.participantColors === "object" ? merged.participantColors : {};
        var cleanParticipantColors = {};
        Object.keys(rawParticipantColors).forEach(function(key) {
          var color = String(rawParticipantColors[key] || "").trim();
          if (/^#[0-9a-f]{6}$/i.test(color)) {
            cleanParticipantColors[String(key)] = color.toLowerCase();
          }
        });
        merged.participantColors = cleanParticipantColors;
        return merged;
      }
      function participantKey(name) {
        return cleanText(name).toLowerCase();
      }
      function participantIdentityKey(participant) {
        participant = participant || {};
        var id = /^\d+$/.test(String(participant.id || "")) ? String(participant.id) : "";
        if (id) {
          return "id:" + id;
        }
        var name = participantKey(participant.name || participant.username || "");
        return name ? "name:" + name : "";
      }
      function normalizeParticipant(participant) {
        if (typeof participant === "string") {
          participant = {
            id: "",
            name: participant
          };
        }
        participant = participant && typeof participant === "object" ? participant : {};
        return {
          id: /^\d+$/.test(String(participant.id || "")) ? String(participant.id) : "",
          name: cleanText(participant.name || participant.username || "")
        };
      }
      function participantColorStorageKey(participant) {
        participant = normalizeParticipant(participant);
        if (participant.id) return "id:" + participant.id;
        var name = participantKey(participant.name);
        return name ? "name:" + name : "";
      }
      function automaticParticipantColor(participant) {
        participant = normalizeParticipant(participant);
        var seed = participant.id || participant.name || "?";
        var hash = 0;
        for (var i = 0; i < seed.length; i++) {
          hash = hash * 31 + seed.charCodeAt(i) >>> 0;
        }
        var palette = [ "#9b5b3e", "#66758f", "#805f79", "#5f7c67", "#98733d", "#8b5963", "#52737a", "#765d96" ];
        return palette[hash % palette.length];
      }
      function participantNameColor(participant, config) {
        participant = normalizeParticipant(participant);
        config = config || currentConfig || DEFAULTS;
        var colors = config.participantColors && typeof config.participantColors === "object" ? config.participantColors : {};
        var idKey = participant.id ? "id:" + participant.id : "";
        var nameKey = participant.name ? "name:" + participantKey(participant.name) : "";
        var value = idKey && colors[idKey] || nameKey && colors[nameKey] || "";
        return /^#[0-9a-f]{6}$/i.test(String(value)) ? String(value) : automaticParticipantColor(participant);
      }
      function participantColorsFromEditor() {
        var result = {};
        $("#sns-participants-editor").children(".sns-participant-settings-row").each(function() {
          var $row = $(this);
          var participant = {
            id: String($row.attr("data-id") || ""),
            name: cleanText($row.attr("data-name"))
          };
          var key = participantColorStorageKey(participant);
          var color = String($row.find(".sns-participant-color").first().val() || "").trim();
          if (key && /^#[0-9a-f]{6}$/i.test(color)) {
            result[key] = color.toLowerCase();
          }
        });
        return result;
      }
      function applyParticipantNameColors(config) {
        config = config || currentConfig || DEFAULTS;
        getTopic().find(".sns-message").each(function() {
          var $post = $(this);
          var participant = {
            id: getPostUserId($post),
            name: getPostAuthor($post)
          };
          $post.find(".sns-author-name").css("color", participantNameColor(participant, config));
        });
      }
      function ownerName() {
        return ownerParticipant().name;
      }
      function collectParticipants() {
        var seen = {};
        var list = [];
        getTopic().find(".post").each(function() {
          var $post = $(this);
          var participant = {
            id: getPostUserId($post),
            name: getPostAuthor($post),
            avatar: getPostAvatar($post)
          };
          if (!participant.name) {
            return;
          }
          var key = participantIdentityKey(participant);
          if (!key || seen[key]) {
            return;
          }
          seen[key] = true;
          list.push(participant);
        });
        return list;
      }
      function knownParticipantFromPosts(participant) {
        participant = normalizeParticipant(participant);
        var result = null;
        getTopic().find(".post").each(function() {
          var $post = $(this);
          var item = {
            id: getPostUserId($post),
            name: getPostAuthor($post),
            avatar: getPostAvatar($post)
          };
          var same = participant.id && item.id && participant.id === item.id || !participant.id && participant.name && participantKey(participant.name) === participantKey(item.name);
          if (!same) {
            return;
          }
          result = item;
          return false;
        });
        return result;
      }
      function knownAvatarForParticipant(participant) {
        var known = knownParticipantFromPosts(participant);
        return known && known.avatar ? known.avatar : "";
      }
      function knownAvatarForName(name) {
        return knownAvatarForParticipant({
          id: "",
          name: name
        });
      }
      function effectiveParticipants(config) {
        config = config || currentConfig || readConfig();
        var owner = ownerParticipant();
        var source = [];
        if (config && config.participantsConfigured) {
          source = [ owner ].concat(Array.isArray(config.participants) ? config.participants : []);
        } else {
          source = collectParticipants();
          if (owner.name || owner.id) {
            source.unshift(owner);
          }
        }
        var seenIds = {};
        var seenNames = {};
        var result = [];
        source.forEach(function(raw) {
          var item = normalizeParticipant(raw);
          if (!item.id && !item.name) {
            return;
          }
          var known = knownParticipantFromPosts(item);
          if (known) {
            if (!item.id && known.id) {
              item.id = known.id;
            }
            if (known.name) {
              item.name = known.name;
            }
          }
          if (item.id && seenIds[item.id]) {
            return;
          }
          var nameKey = participantKey(item.name);
          if (!item.id && nameKey && seenNames[nameKey]) {
            return;
          }
          if (item.id) {
            seenIds[item.id] = true;
          }
          if (nameKey) {
            seenNames[nameKey] = true;
          }
          result.push(item);
        });
        return result;
      }
      function effectiveParticipantNames(config) {
        return effectiveParticipants(config).map(function(item) {
          return item.name;
        }).filter(Boolean);
      }
      function isParticipantIdentity(name, id, config) {
        name = cleanText(name);
        id = /^\d+$/.test(String(id || "")) ? String(id) : "";
        var nameKey = participantKey(name);
        return effectiveParticipants(config).some(function(participant) {
          if (id && participant.id) {
            return id === participant.id;
          }
          return !!nameKey && participantKey(participant.name) === nameKey;
        });
      }
      function isParticipantName(name, config) {
        return isParticipantIdentity(name, "", config);
      }
      function isOwner() {
        var owner = ownerParticipant();
        var meId = getCurrentUserId();
        if (owner.id && meId) {
          return owner.id === meId;
        }
        return !!owner.name && !!getCurrentUser() && participantKey(owner.name) === participantKey(getCurrentUser());
      }
      function isCurrentParticipant(config) {
        var me = getCurrentUser();
        var meId = getCurrentUserId();
        return !!me && isParticipantIdentity(me, meId, config);
      }
      function canEditAppearance(config) {
        return isCurrentParticipant(config);
      }
      function participantProfileCacheKey(participant) {
        participant = normalizeParticipant(participant);
        return "SNS_PARTICIPANT_PROFILE_V58_" + (participant.id ? "ID_" + participant.id : "NAME_" + participantKey(participant.name));
      }
      function readParticipantProfileCache(participant) {
        try {
          var raw = localStorage.getItem(participantProfileCacheKey(participant));
          if (!raw) {
            return null;
          }
          var data = JSON.parse(raw);
          if (!data || !data.savedAt || Date.now() - data.savedAt > 12 * 60 * 60 * 1e3) {
            return null;
          }
          return data;
        } catch (error) {
          return null;
        }
      }
      function saveParticipantProfileCache(participant, data) {
        try {
          localStorage.setItem(participantProfileCacheKey(participant), JSON.stringify($.extend({
            savedAt: Date.now()
          }, data || {})));
        } catch (error) {}
      }
      function profileAvatarFromHtml(html) {
        var $doc = $("<div></div>").append($.parseHTML(String(html || ""), document, false));
        var selectors = [ "#pun-profile #viewprofile img.avatardemo", "#pun-profile img.avatardemo", "#viewprofile img.avatardemo", "#viewprofile-next img.avatardemo", "img.avatardemo", "#viewprofile .pa-avatar img", ".pa-avatar img", ".profile-avatar img", ".avatar img", "img.avatar", "#profile-left img", ".profile-left img" ];
        for (var i = 0; i < selectors.length; i++) {
          var src = $doc.find(selectors[i]).first().attr("src");
          if (src) {
            return src;
          }
        }
        var fallback = "";
        $doc.find("img[src]").each(function() {
          var src = $(this).attr("src") || "";
          if (/avatar|avatars/i.test(src)) {
            fallback = src;
            return false;
          }
        });
        return fallback;
      }
      function profileNameFromHtml(html, fallback) {
        var $doc = $("<div></div>").append($.parseHTML(String(html || ""), document, false));
        function normalizeProfileName(value) {
          value = cleanText(value);
          value = value.replace(/^(?:\u043f\u0440\u043e\u0444\u0438\u043b\u044c|profile)\s*:\s*/i, "");
          value = value.replace(/^(?:\u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440\s+\u043f\u0440\u043e\u0444\u0438\u043b\u044f|view\s+profile)\s*:\s*/i, "");
          return cleanText(value);
        }
        var selectors = [ "#viewprofile h2 span", "#viewprofile-next h2 span", "#viewprofile h1 span", "#viewprofile-next h1 span", "#profile-left h1", "#profile-left h2", "#profile-left strong", ".profile-left h1", ".profile-left h2", ".profile-left strong", ".profile h1", ".profile h2" ];
        for (var i = 0; i < selectors.length; i++) {
          var name = normalizeProfileName($doc.find(selectors[i]).first().text());
          if (name && name.length <= 100) {
            return name;
          }
        }
        var title = cleanText($doc.find("title").first().text());
        if (title) {
          var pieces = title.split(/[-|]/);
          for (var p = 0; p < pieces.length; p++) {
            var candidate = normalizeProfileName(pieces[p]);
            if (candidate && candidate.length <= 100 && !/^(?:test|forum|rusff)$/i.test(candidate)) {
              return candidate;
            }
          }
        }
        return normalizeProfileName(fallback);
      }
      function profileIdFromUrl(url) {
        return profileIdFromHref(url);
      }
      var participantLookupPending = {};
      var participantRecentResult = {};
      function resolveParticipantProfile(participant, callback) {
        participant = normalizeParticipant(participant);
        if (!participant.id && !participant.name) {
          if (callback) {
            callback(null);
          }
          return;
        }
        var known = knownParticipantFromPosts(participant);
        if (known && known.id && known.avatar) {
          if (callback) {
            callback({
              found: true,
              id: known.id,
              name: known.name || participant.name,
              avatar: known.avatar
            });
          }
          return;
        }
        if (known && known.id) {
          participant.id = known.id;
          if (known.name) {
            participant.name = known.name;
          }
        }
        var cached = readParticipantProfileCache(participant);
        if (cached && cached.found !== false && String(cached.avatar || "").trim()) {
          if (callback) {
            callback(cached);
          }
          return;
        }
        var pendingKey = participant.id ? "id:" + participant.id : "name:" + participantKey(participant.name);
        var recent = participantRecentResult[pendingKey];
        if (recent && Date.now() - recent.at < 6e4) {
          if (callback) callback(recent.data);
          return;
        }
        if (participantLookupPending[pendingKey]) {
          participantLookupPending[pendingKey].push(callback);
          return;
        }
        participantLookupPending[pendingKey] = [ callback ];
        function finish(data) {
          participantRecentResult[pendingKey] = {
            at: Date.now(),
            data: data || null
          };
          var callbacks = participantLookupPending[pendingKey] || [];
          delete participantLookupPending[pendingKey];
          if (data && String(data.avatar || "").trim()) {
            saveParticipantProfileCache({
              id: data.id || participant.id,
              name: data.name || participant.name
            }, data);
          }
          callbacks.forEach(function(fn) {
            if (typeof fn === "function") {
              fn(data || null);
            }
          });
        }
        function loadProfile(profileUrl, expectedName, expectedId) {
          var id = String(expectedId || profileIdFromUrl(profileUrl) || participant.id || "");
          if (!/^\d+$/.test(id)) {
            finish({
              found: false,
              id: "",
              name: expectedName || participant.name || "",
              avatar: ""
            });
            return;
          }
          var apiUrl = new URL("api.php", location.href);
          apiUrl.searchParams.set("method", "users.get");
          apiUrl.searchParams.set("user_id", id);
          apiUrl.searchParams.set("fields", "user_id,username,avatar");
          apiUrl.searchParams.set("_sns", String(Date.now()));
          var finished = false;
          function finishOnce(data) {
            if (finished) {
              return;
            }
            finished = true;
            finish(data);
          }
          var watchdog = null;
          SNSRequest({
            url: apiUrl.toString(),
            type: "GET",
            dataType: "json",
            cache: false,
            timeout: 4500
          }).done(function(json) {
            clearTimeout(watchdog);
            var users = json && json.response && json.response.users ? json.response.users : [];
            var user = null;
            if (Array.isArray(users)) {
              for (var i = 0; i < users.length; i++) {
                if (String(users[i].user_id || users[i].id || "") === id) {
                  user = users[i];
                  break;
                }
              }
              if (!user && users.length === 1) {
                user = users[0];
              }
            } else if (users && typeof users === "object") {
              if (users[id]) {
                user = users[id];
              } else {
                for (var key in users) {
                  if (!Object.prototype.hasOwnProperty.call(users, key)) {
                    continue;
                  }
                  var candidate = users[key];
                  if (String(candidate.user_id || candidate.id || key || "") === id) {
                    user = candidate;
                    break;
                  }
                }
              }
            }
            if (!user) {
              finishOnce({
                found: true,
                id: id,
                name: expectedName || participant.name || "",
                avatar: ""
              });
              return;
            }
            finishOnce({
              found: true,
              id: String(user.user_id || user.id || id),
              name: cleanText(user.username || expectedName || participant.name || ""),
              profileUrl: new URL("profile.php?id=" + encodeURIComponent(id), location.href).toString(),
              avatar: function() {
                var raw = String(user.avatar || "").trim();
                if (!raw) {
                  return "";
                }
                try {
                  return new URL(raw, location.href).toString();
                } catch (error) {
                  return raw;
                }
              }()
            });
          }).fail(function() {
            clearTimeout(watchdog);
            finishOnce({
              found: true,
              id: id,
              name: expectedName || participant.name || "",
              avatar: ""
            });
          });
        }
        if (participant.id) {
          var directProfileUrl = new URL("profile.php", location.href);
          directProfileUrl.searchParams.set("id", participant.id);
          loadProfile(directProfileUrl.toString(), participant.name, participant.id);
          return;
        }
        var userListUrl = new URL("userlist.php", location.href);
        userListUrl.searchParams.set("username", participant.name);
        userListUrl.searchParams.set("show_group", "-1");
        SNSRequest({
          url: userListUrl.toString(),
          type: "GET",
          dataType: "html",
          cache: false,
          timeout: 5e3
        }).done(function(html) {
          var $doc = $("<div></div>").append($.parseHTML(String(html || ""), document, false));
          var wantedKey = participantKey(participant.name);
          var $profile = $();
          $doc.find('a[href*="profile.php?id="]').each(function() {
            var $link = $(this);
            if (participantKey($link.text()) === wantedKey) {
              $profile = $link;
              return false;
            }
          });
          if (!$profile.length) {
            finish({
              found: false,
              id: "",
              name: participant.name,
              avatar: ""
            });
            return;
          }
          var href = $profile.attr("href") || "";
          var id = profileIdFromHref(href);
          if (!href || !id) {
            finish({
              found: false,
              id: "",
              name: participant.name,
              avatar: ""
            });
            return;
          }
          var canonicalName = cleanText($profile.text()) || participant.name;
          var profileUrl = new URL(href, location.href).toString();
          loadProfile(profileUrl, canonicalName, id);
        }).fail(function() {
          finish({
            found: false,
            id: "",
            name: participant.name,
            avatar: ""
          });
        });
      }
      window.SNSResolveUserVisual = function(userId, username, callback) {
        resolveParticipantProfile({
          id: String(userId || ""),
          name: cleanText(username || "")
        }, callback);
      };
      $(document).off("sns_user_visual_loaded.snsV58").on("sns_user_visual_loaded.snsV58", function() {
        if (window.SNSEnhancePosts) window.SNSEnhancePosts();
        renderHeaderTools();
      });
      function ensureNoiseLayer() {
        var $shell = $("#sns-chat-shell");
        if (!$shell.length) {
          return $();
        }
        var $noise = $shell.children(".sns-bg-noise");
        if (!$noise.length) {
          $noise = $('<div class="sns-bg-noise" aria-hidden="true"></div>');
          $shell.prepend($noise);
        }
        return $noise;
      }
      function ensureDialogBackgroundLayer() {
        var $shell = $("#sns-chat-shell");
        var $topic = getTopic();
        if (!$shell.length || !$topic.length) {
          return $();
        }
        var $layer = $shell.children(".sns-dialog-bg-layer").first();
        if (!$layer.length) {
          $layer = $('<div class="sns-dialog-bg-layer" aria-hidden="true">' + '<div class="sns-dialog-bg-image"></div>' + "</div>");
          $topic.before($layer);
        }
        return $layer;
      }
      function positionDialogBackgroundLayer() {
        var $shell = $("#sns-chat-shell");
        var $topic = getTopic();
        var $layer = ensureDialogBackgroundLayer();
        if (!$shell.length || !$topic.length || !$layer.length) {
          return;
        }
        var shellRect = $shell.get(0).getBoundingClientRect();
        var topicRect = $topic.get(0).getBoundingClientRect();
        var left = topicRect.left - shellRect.left;
        var top = topicRect.top - shellRect.top;
        $layer.css({
          left: Math.round(left) + "px",
          top: Math.round(top) + "px",
          width: Math.round(topicRect.width) + "px",
          height: Math.round(topicRect.height) + "px"
        });
      }
      function addCustomStyle() {
        if (document.getElementById(CUSTOM_STYLE_ID)) return;
        var style = document.createElement("style");
        style.id = CUSTOM_STYLE_ID;
        style.textContent = [ "body.sns-chat-page #pun-viewtopic,", "body.sns-chat-page #pun-viewtopic>.main,", "body.sns-chat-page #pun-viewtopic .main-content{overflow:visible!important;}", "body.sns-chat-page #sns-chat-shell{", "position:relative!important;", "box-sizing:border-box!important;", "overflow:visible!important;", "padding:0 0 38px!important;", "background:none!important;", "border-radius:0!important;", "isolation:isolate;", "--sns-geo-offset:82px;", "--sns-card-offset:108px;", "--sns-card-overhang:12px;", "--sns-geo-bottom:38px;", "--sns-bg-height:722px;", "}", "body.sns-chat-page #sns-chat-shell:before{", 'content:"";', "position:absolute;", "left:0;top:0;", "width:calc(100% - var(--sns-geo-offset));", "height:var(--sns-bg-height);", "z-index:0;", "background-color:#a97957;", "background-image:var(--sns-bg-image,none);", "background-size:cover;", "background-position:center;", "background-repeat:no-repeat;", "border-radius:0;", "pointer-events:none;", "}", "body.sns-chat-page #sns-chat-shell:after{", 'content:"";', "position:absolute;", "left:0;top:0;", "width:calc(100% - var(--sns-geo-offset));", "height:var(--sns-bg-height);", "z-index:1;", "background:var(--sns-overlay-color,#714b35);", "opacity:var(--sns-overlay-opacity,.36);", "border-radius:0;", "pointer-events:none;", "}", "body.sns-chat-page #sns-chat-shell>.sns-bg-noise{", "position:absolute;", "left:0;top:0;", "width:calc(100% - var(--sns-geo-offset));", "height:var(--sns-bg-height);", "z-index:2;", "pointer-events:none;", "opacity:var(--sns-noise-opacity,0);", "mix-blend-mode:overlay;", "filter:contrast(175%);", 'background-image:url("data:image/svg+xml,%3Csvg xmlns=%22http://www.w3.org/2000/svg%22 width=%22180%22 height=%22180%22 viewBox=%220 0 180 180%22%3E%3Cfilter id=%22n%22 x=%220%22 y=%220%22 width=%22100%25%22 height=%22100%25%22%3E%3CfeTurbulence type=%22fractalNoise%22 baseFrequency=%220.78%22 numOctaves=%224%22 stitchTiles=%22stitch%22/%3E%3C/filter%3E%3Crect width=%22100%25%22 height=%22100%25%22 filter=%22url(%23n)%22 opacity=%221%22/%3E%3C/svg%3E");', "background-repeat:repeat;", "background-size:180px 180px;", "}", "body.sns-chat-page #sns-chat-header{", "position:relative!important;", "z-index:3!important;", "box-sizing:border-box!important;", "width:calc(100% - var(--sns-geo-offset))!important;", "min-height:176px!important;", "margin:0 var(--sns-geo-offset) 0 0!important;", "padding:34px 30px 64px var(--sns-geo-offset)!important;", "background:transparent!important;", "border-radius:0!important;", "align-items:flex-start!important;", "}", "body.sns-chat-page #sns-chat-header .sns-head-left{align-items:center!important;gap:14px!important;}", "body.sns-chat-page #sns-chat-header .sns-head-avatar-box{", "position:relative!important;", "display:block!important;", "flex:0 0 54px!important;", "width:54px!important;", "height:54px!important;", "border-radius:50%!important;", "overflow:hidden!important;", "box-shadow:0 4px 18px rgba(0,0,0,.13)!important;", "isolation:isolate!important;", "}", "body.sns-chat-page #sns-chat-header .sns-head-avatar{", "position:relative!important;", "z-index:1!important;", "display:block!important;", "box-sizing:border-box!important;", "width:54px!important;height:54px!important;flex-basis:54px!important;", "border:3px solid rgba(255,255,255,.78)!important;", "border-radius:50%!important;", "box-shadow:none!important;", "filter:grayscale(var(--sns-avatar-gray,0%))!important;", "}", "body.sns-chat-page #sns-chat-header .sns-head-avatar-box:before{", 'content:"";', "position:absolute;", "z-index:2;", "inset:3px;", "pointer-events:none;", "border-radius:50%;", "background:var(--sns-avatar-tint,#8a5a3a);", "opacity:var(--sns-avatar-tint-opacity,0);", "mix-blend-mode:var(--sns-avatar-blend,normal);", "}", "body.sns-chat-page #sns-chat-header .sns-head-avatar-box:after{", 'content:"";', "position:absolute;", "z-index:3;", "inset:3px;", "pointer-events:none;", "border-radius:50%;", "opacity:var(--sns-avatar-noise-opacity,0);", "mix-blend-mode:overlay;", "filter:contrast(165%);", 'background-image:url("data:image/svg+xml,%3Csvg xmlns=%22http://www.w3.org/2000/svg%22 width=%2290%22 height=%2290%22 viewBox=%220 0 90 90%22%3E%3Cfilter id=%22n%22%3E%3CfeTurbulence type=%22fractalNoise%22 baseFrequency=%220.86%22 numOctaves=%224%22 stitchTiles=%22stitch%22/%3E%3C/filter%3E%3Crect width=%22100%25%22 height=%22100%25%22 filter=%22url(%23n)%22 opacity=%221%22/%3E%3C/svg%3E");', "background-size:90px 90px;", "}", "body.sns-chat-page #sns-chat-header .sns-head-title{", "font:700 21px/1.2 Arial,sans-serif!important;", "color:#fff!important;", "text-shadow:0 1px 8px rgba(0,0,0,.14);", "}", "body.sns-chat-page #sns-chat-header .sns-head-subtitle{", "margin-top:5px!important;color:rgba(255,255,255,.75)!important;font-size:10px!important;", "}", "body.sns-chat-page #sns-chat-header .sns-head-badge{display:none!important;}", "body.sns-chat-page .sns-head-tools{", "display:flex;flex:0 0 auto;align-items:center;gap:9px;margin-left:auto;", "}", "body.sns-chat-page #sns-chat-header .sns-head-left{flex:1 1 auto;min-width:0!important;}", "body.sns-chat-page .sns-head-participants{", "display:flex;align-items:center;gap:6px;", "max-width:min(470px,58vw);", "overflow-x:auto;overflow-y:hidden;", "scrollbar-width:none;-ms-overflow-style:none;", "padding:2px 1px;", "}", "body.sns-chat-page .sns-head-participants::-webkit-scrollbar{display:none;}", "body.sns-chat-page .sns-participant-more{", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:38px;height:38px;border-radius:50%;box-sizing:border-box;", "border:2px solid rgba(255,255,255,.8);background:rgba(255,255,255,.20);", "color:#fff;font:700 10px Arial,sans-serif;box-shadow:0 2px 10px rgba(0,0,0,.12);", "backdrop-filter:blur(5px);", "}", "body.sns-chat-page .sns-participant-avatar{", "display:block;flex:0 0 38px;width:38px;height:38px;object-fit:cover;border-radius:50%;", "border:2px solid rgba(255,255,255,.8);box-sizing:border-box;", "box-shadow:0 2px 10px rgba(0,0,0,.12);background:#fff;", "}", "body.sns-chat-page .sns-participant-fallback{", "display:flex;align-items:center;justify-content:center;color:#555;font:700 11px Arial,sans-serif;", "}", "body.sns-chat-page .sns-settings-open,body.sns-chat-page .sns-search-open{", "display:flex!important;align-items:center!important;justify-content:center!important;", "flex:0 0 38px;width:38px;height:38px;padding:0!important;", "border:0;border-radius:50%;cursor:pointer;", "background:rgba(255,255,255,.18);color:#fff;", "backdrop-filter:blur(5px);-webkit-backdrop-filter:blur(5px);", "}", "body.sns-chat-page .sns-search-open svg{", "display:block!important;width:18px!important;height:18px!important;", "fill:none!important;stroke:currentColor!important;stroke-width:2.15!important;", "stroke-linecap:round!important;stroke-linejoin:round!important;", "pointer-events:none!important;", "}", "body.sns-chat-page .sns-search-open:hover{background:rgba(255,255,255,.28);}", "body.sns-chat-page .sns-settings-icon{", "display:block!important;width:19px!important;height:19px!important;", "fill:none!important;stroke:currentColor!important;stroke-width:2.15!important;", "stroke-linecap:round!important;stroke-linejoin:round!important;", "overflow:visible!important;pointer-events:none!important;", "transform-box:fill-box!important;", "transform-origin:center!important;", "will-change:transform;", "}", "body.sns-chat-page .sns-settings-open:hover{background:rgba(255,255,255,.28);}", "body.sns-chat-page .sns-settings-open.is-spinning{", "background:rgba(255,255,255,.30)!important;", "}", "body.sns-chat-page .sns-settings-open.is-spinning .sns-settings-icon{", "animation:snsSettingsGearSpin .52s cubic-bezier(.22,.72,.22,1) both!important;", "}", "@keyframes snsSettingsGearSpin{", "0%{transform:rotate(0deg) scale(1);}", "35%{transform:rotate(190deg) scale(.88);}", "72%{transform:rotate(425deg) scale(1.08);}", "100%{transform:rotate(540deg) scale(1);}", "}", "body.sns-chat-page #sns-chat-shell>.topic{", "position:relative!important;z-index:5!important;", "-webkit-overflow-scrolling:touch!important;", "overscroll-behavior:auto!important;", "touch-action:pan-y!important;", "box-sizing:border-box!important;", "width:calc(100% - var(--sns-card-offset) + var(--sns-card-overhang))!important;height:625px!important;", "margin:-82px 0 0 var(--sns-card-offset)!important;padding:54px 36px 38px!important;", "scroll-padding-top:18px!important;", "background:transparent!important;", "box-shadow:", "0 -4px 11px rgba(18,18,22,.10),", "0 -15px 32px rgba(18,20,28,.14),", "0 3px 9px rgba(20,18,16,.10),", "0 16px 34px rgba(20,22,30,.18),", "0 34px 78px rgba(20,22,30,.22)!important;", "border-radius:16px 16px 0 0!important;", "}", "body.sns-chat-page #sns-chat-shell>.sns-dialog-bg-layer{", "position:absolute!important;", "z-index:4!important;", "display:block!important;", "box-sizing:border-box!important;", "margin:0!important;", "overflow:hidden!important;", "pointer-events:none!important;", "border-radius:16px 16px 0 0!important;", "box-shadow:0 -10px 24px rgba(18,20,28,.13),0 -24px 48px rgba(18,20,28,.11),0 18px 48px rgba(18,20,28,.16),0 42px 96px rgba(18,20,28,.18)!important;", "background:var(--sns-dialog-color,#fbfaf7)!important;", "}", "body.sns-chat-page #sns-chat-shell>.sns-dialog-bg-layer>.sns-dialog-bg-image{", "position:absolute!important;", "inset:-26px!important;", "background-color:var(--sns-dialog-color,#fbfaf7)!important;", "background-image:var(--sns-dialog-image,none)!important;", "background-size:var(--sns-dialog-size,cover)!important;", "background-repeat:var(--sns-dialog-repeat,no-repeat)!important;", "background-position:var(--sns-dialog-position,center)!important;", "filter:blur(var(--sns-dialog-blur,0px))!important;", "transform:scale(1.04)!important;", "transform-origin:center!important;", "}", "body.sns-chat-page #sns-chat-shell>.sns-dialog-bg-layer:after{", 'content:"";', "position:absolute;", "inset:0;", "pointer-events:none;", "opacity:var(--sns-dialog-noise-opacity,0);", "mix-blend-mode:overlay;", "filter:contrast(175%);", 'background-image:url("data:image/svg+xml,%3Csvg xmlns=%22http://www.w3.org/2000/svg%22 width=%22160%22 height=%22160%22 viewBox=%220 0 160 160%22%3E%3Cfilter id=%22n%22%3E%3CfeTurbulence type=%22fractalNoise%22 baseFrequency=%220.78%22 numOctaves=%224%22 stitchTiles=%22stitch%22/%3E%3C/filter%3E%3Crect width=%22100%25%22 height=%22100%25%22 filter=%22url(%23n)%22 opacity=%221%22/%3E%3C/svg%3E");', "background-size:160px 160px;", "background-repeat:repeat;", "}", "body.sns-chat-page #sns-chat-shell>.topic>.post,", "body.sns-chat-page #sns-chat-shell>.topic>#sns-empty{", "position:relative!important;", "z-index:1!important;", "}", "body.sns-chat-page #sns-chat-shell>.topic>.post:first-of-type{margin-top:8px!important;}", "body.sns-chat-page #sns-chat-shell>.topic>.post.sns-menu-active{", "z-index:10000!important;", "}", "body.sns-chat-page #sns-chat-shell>.topic>.post.sns-menu-active .sns-controls{", "z-index:10001!important;", "opacity:1!important;", "}", "body.sns-chat-page #sns-chat-shell>.topic>.post.sns-menu-active .sns-menu{", "z-index:10002!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-own .post-content{", "background:linear-gradient(var(--sns-gradient-angle,135deg),var(--sns-own-g1,#b65f3a),var(--sns-own-g2,#d7a44a))!important;", "color:#fff!important;border-radius:22px 22px 5px 22px!important;", "box-shadow:0 3px 10px rgba(81,116,210,.10);", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-other .post-content{", "background:var(--sns-other-bubble,#eee7dd)!important;", "color:var(--sns-other-text,#333)!important;border-radius:22px 22px 22px 5px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-other .post-content a:not(.sns-reply-preview){color:inherit!important;}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-other .post-content .sns-audio-card,body.sns-chat-page #pun-viewtopic .sns-message.sns-other .post-content .sns-voice-card{color:inherit!important;}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-has-media .post-body{", "max-width:400px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-only .post-body{", "width:min(380px,70%)!important;max-width:380px!important;min-width:0!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-only .post-box{", "width:100%!important;max-width:380px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-only .post-content{", "box-sizing:border-box!important;width:100%!important;max-width:380px!important;", "padding:5px!important;box-shadow:none!important;overflow:hidden!important;", "border-radius:18px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-only.sns-own .post-content{", "background:linear-gradient(var(--sns-gradient-angle,135deg),var(--sns-own-g1,#b65f3a),var(--sns-own-g2,#d7a44a))!important;", "border-radius:18px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-only.sns-other .post-content{", "background:var(--sns-other-bubble,#eee7dd)!important;", "border-radius:18px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-only .post-content p{", "margin:0!important;padding:0!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-only .post-content>a{", "display:block!important;margin:0!important;padding:0!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-only .post-content>img,", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-only .post-content>a>img{", "display:block!important;width:100%!important;max-width:none!important;height:auto!important;", "margin:0!important;padding:0!important;border-radius:13px!important;object-fit:cover!important;", "cursor:zoom-in!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-caption .post-content{", "box-sizing:border-box!important;width:100%!important;max-width:400px!important;padding:6px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-caption .post-content p{", "margin:4px 7px 2px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-caption .post-content p:first-of-type{", "margin-top:4px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-caption .sns-source-media-hidden+br{", "display:none!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-caption .post-content>img,", "body.sns-chat-page #pun-viewtopic .sns-message.sns-media-caption .post-content>a>img{", "display:block!important;width:100%!important;max-width:none!important;height:auto!important;", "margin:0!important;border-radius:15px!important;object-fit:cover!important;", "}", "body.sns-chat-page .sns-media-grid{", "display:grid!important;width:100%!important;", "gap:3px!important;margin:0!important;padding:0!important;", "overflow:hidden!important;border-radius:13px!important;", "background:transparent!important;", "}", "body.sns-chat-page .sns-source-media-hidden{display:none!important;}", "body.sns-chat-page .sns-media-grid .sns-media-item{", "position:relative!important;min-width:0!important;", "overflow:hidden!important;background:transparent!important;", "font-size:0!important;line-height:0!important;", "}", "body.sns-chat-page .sns-media-grid .sns-media-item>a,", "body.sns-chat-page .sns-media-grid .sns-media-item .sns-media-link{", "display:block!important;width:100%!important;height:100%!important;", "margin:0!important;padding:0!important;border:0!important;", "font-size:0!important;line-height:0!important;", "background:transparent!important;", "}", "body.sns-chat-page .sns-media-grid img{", "display:block!important;visibility:visible!important;opacity:1!important;", "width:100%!important;height:100%!important;", "min-width:100%!important;min-height:100%!important;", "max-width:none!important;max-height:none!important;", "margin:0!important;padding:0!important;border:0!important;border-radius:0!important;", "vertical-align:top!important;background:transparent!important;", "object-fit:cover!important;object-position:center!important;", "cursor:zoom-in!important;", "}", "body.sns-chat-page .sns-media-count-2{grid-template-columns:1fr 1fr!important;}", "body.sns-chat-page .sns-media-count-2 .sns-media-item{height:220px!important;}", "body.sns-chat-page .sns-media-count-3{", "grid-template-columns:1.15fr .85fr!important;", "grid-template-rows:128px 128px!important;", "}", "body.sns-chat-page .sns-media-count-3 .sns-media-item:nth-child(1){grid-row:1/3!important;}", "body.sns-chat-page .sns-media-count-3 .sns-media-item:nth-child(1){height:259px!important;}", "body.sns-chat-page .sns-media-count-3 .sns-media-item:nth-child(n+2){height:128px!important;}", "body.sns-chat-page .sns-media-count-4{grid-template-columns:1fr 1fr!important;}", "body.sns-chat-page .sns-media-count-4 .sns-media-item{height:170px!important;}", "body.sns-chat-page .sns-media-count-5{grid-template-columns:repeat(6,1fr)!important;}", "body.sns-chat-page .sns-media-count-5 .sns-media-item:nth-child(1),", "body.sns-chat-page .sns-media-count-5 .sns-media-item:nth-child(2){grid-column:span 3!important;height:170px!important;}", "body.sns-chat-page .sns-media-count-5 .sns-media-item:nth-child(n+3){grid-column:span 2!important;height:135px!important;}", "body.sns-chat-page .sns-media-count-6{grid-template-columns:repeat(6,1fr)!important;}", "body.sns-chat-page .sns-media-count-6 .sns-media-item:nth-child(-n+3){grid-column:span 2!important;height:122px!important;}", "body.sns-chat-page .sns-media-count-6 .sns-media-item:nth-child(4){grid-column:1/-1!important;height:205px!important;}", "body.sns-chat-page .sns-media-count-6 .sns-media-item:nth-child(5){grid-column:span 4!important;height:145px!important;}", "body.sns-chat-page .sns-media-count-6 .sns-media-item:nth-child(6){grid-column:span 2!important;height:145px!important;}", "body.sns-chat-page .sns-media-count-7{grid-template-columns:repeat(6,1fr)!important;}", "body.sns-chat-page .sns-media-count-7 .sns-media-item:nth-child(-n+3){grid-column:span 2!important;height:120px!important;}", "body.sns-chat-page .sns-media-count-7 .sns-media-item:nth-child(n+4){grid-column:span 3!important;height:145px!important;}", "body.sns-chat-page .sns-media-count-8{grid-template-columns:repeat(6,1fr)!important;}", "body.sns-chat-page .sns-media-count-8 .sns-media-item:nth-child(-n+2){grid-column:span 3!important;height:155px!important;}", "body.sns-chat-page .sns-media-count-8 .sns-media-item:nth-child(n+3){grid-column:span 2!important;height:125px!important;}", "body.sns-chat-page .sns-media-count-9{grid-template-columns:repeat(3,1fr)!important;}", "body.sns-chat-page .sns-media-count-9 .sns-media-item{height:125px!important;}", "body.sns-chat-page .sns-media-count-many{grid-template-columns:repeat(3,1fr)!important;}", "body.sns-chat-page .sns-media-count-many .sns-media-item{height:120px!important;}", "body.sns-viewer-open{overflow:hidden!important;}", "#sns-photo-viewer{", "position:fixed!important;inset:0!important;z-index:2147483000!important;", "display:none;align-items:center;justify-content:center;", "background:rgba(8,8,10,.9);backdrop-filter:blur(8px);", "}", "#sns-photo-viewer.is-open{display:flex!important;}", "#sns-photo-viewer .sns-viewer-stage{", "display:flex;align-items:center;justify-content:center;", "width:min(86vw,1180px);height:88vh;padding:24px;box-sizing:border-box;", "}", "#sns-photo-viewer .sns-viewer-image{", "display:block;max-width:100%;max-height:100%;width:auto;height:auto;", "object-fit:contain;border-radius:8px;", "}", "#sns-photo-viewer button{", "position:absolute;z-index:2;border:0;cursor:pointer;", "display:flex;align-items:center;justify-content:center;", "background:rgba(255,255,255,.12);color:#fff;", "}", "#sns-photo-viewer .sns-viewer-close{", "right:22px;top:20px;width:44px;height:44px;border-radius:50%;font:300 30px/1 Arial,sans-serif;", "}", "#sns-photo-viewer .sns-viewer-prev,#sns-photo-viewer .sns-viewer-next{", "top:50%;transform:translateY(-50%);width:48px;height:68px;border-radius:12px;font:300 46px/1 Arial,sans-serif;", "}", "#sns-photo-viewer .sns-viewer-prev{left:20px;}", "#sns-photo-viewer .sns-viewer-next{right:20px;}", "#sns-photo-viewer .sns-viewer-count{", "position:absolute;left:50%;bottom:20px;transform:translateX(-50%);", "padding:7px 12px;border-radius:20px;background:rgba(0,0,0,.42);", "color:#fff;font:11px Arial,sans-serif;", "}", "body.sns-chat-page #pun-viewtopic .sns-mini-avatar{width:36px!important;height:36px!important;left:0!important;}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-other .post-body{margin-left:48px!important;}", "body.sns-chat-page #pun-viewtopic .sns-meta{font-size:9px!important;color:#9a9aa2!important;}", "body.sns-chat-page #pun-viewtopic .sns-config-post{display:none!important;}", "body.sns-chat-page #sns-composer-ui{", "position:relative!important;z-index:20!important;", "box-sizing:border-box!important;", "display:grid!important;", "grid-template-columns:34px 34px minmax(0,1fr) 42px!important;", "grid-template-rows:auto!important;", "align-items:center!important;column-gap:10px!important;", "width:calc(100% - var(--sns-card-offset) + var(--sns-card-overhang))!important;", "margin:0 0 0 var(--sns-card-offset)!important;padding:12px 16px!important;", "background:rgba(240,240,243,.98)!important;border:0!important;border-radius:0 0 16px 16px!important;", "box-shadow:", "0 8px 18px rgba(20,18,16,.10),", "0 24px 52px rgba(20,22,30,.20),", "0 42px 88px rgba(20,22,30,.22),", "inset 0 1px 0 rgba(255,255,255,.68)!important;", "}", "body.sns-chat-page #sns-composer-ui>.sns-ui-plus{grid-column:1!important;}", "body.sns-chat-page #sns-composer-ui>.sns-ui-format{grid-column:2!important;}", "body.sns-chat-page #sns-composer-ui>.sns-ui-input-wrap{grid-column:3!important;}", "body.sns-chat-page #sns-composer-ui>.sns-ui-send{grid-column:4!important;}", "body.sns-chat-page .sns-ui-input-wrap{", "display:block!important;", "box-sizing:border-box!important;", "width:100%!important;min-width:0!important;max-width:none!important;", "margin:0!important;padding:0!important;", "}", "body.sns-chat-page #sns-ui-input{", "display:block!important;", "box-sizing:border-box!important;width:100%!important;min-width:0!important;max-width:none!important;", "height:42px!important;min-height:42px!important;", "margin:0!important;", "background:#fff!important;color:#333!important;border:0!important;border-radius:7px!important;", "padding:11px 14px!important;line-height:20px!important;", "box-shadow:inset 0 0 0 1px rgba(0,0,0,.04)!important;text-align:left!important;", "}", "body.sns-chat-page #sns-pending-stack{", "display:flex!important;flex-direction:column!important;align-items:flex-end!important;", "gap:10px!important;width:100%!important;margin:0!important;padding:0!important;", "box-sizing:border-box!important;pointer-events:none!important;", "}", "body.sns-chat-page #sns-pending-stack .sns-pending-row{", "display:flex!important;flex-direction:column!important;align-items:flex-end!important;", "width:auto!important;max-width:min(74%,430px)!important;", "margin:0!important;padding:0!important;", "opacity:.62!important;transition:opacity .16s ease,transform .16s ease!important;", "}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.is-sending{opacity:.78!important;}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.is-server-confirmed{opacity:1!important;}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.is-server-confirmed .sns-pending-meta{display:none!important;}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.is-confirmed{opacity:0!important;transform:translateY(3px)!important;}", "body.sns-chat-page #sns-pending-stack .sns-pending-meta{", "margin:0 4px 4px!important;color:rgba(255,255,255,.58)!important;", "font:8px/1.2 Arial,sans-serif!important;text-align:right!important;", "}", "body.sns-chat-page #sns-pending-stack .sns-pending-bubble{", "box-sizing:border-box!important;width:auto!important;max-width:100%!important;", "padding:10px 13px!important;", "background:linear-gradient(var(--sns-gradient-angle,135deg),var(--sns-own-g1,#b65f3a),var(--sns-own-g2,#d7a44a))!important;", "color:#fff!important;border:0!important;border-radius:22px 22px 5px 22px!important;", "box-shadow:0 3px 10px rgba(81,116,210,.08)!important;", "font:13px/1.45 Arial,sans-serif!important;text-align:left!important;", "white-space:pre-wrap!important;overflow-wrap:anywhere!important;", "}", "@media (max-width:650px){", "body.sns-chat-page #sns-pending-stack .sns-pending-row{max-width:84%!important;}", "}", "body.sns-chat-page #sns-ui-input::placeholder{color:var(--sns-own-g1,#b65f3a)!important;opacity:.72;text-align:left!important;}", "body.sns-chat-page .sns-ui-plus,body.sns-chat-page .sns-ui-format,body.sns-chat-page .sns-ui-send{", "display:flex!important;align-items:center!important;justify-content:center!important;", "margin:0!important;padding:0!important;line-height:1!important;", "border:0!important;border-radius:50%!important;box-sizing:border-box!important;", "align-self:center!important;", "}", "body.sns-chat-page .sns-ui-plus,body.sns-chat-page .sns-ui-format{", "flex:0 0 34px!important;width:34px!important;height:34px!important;", "}", "body.sns-chat-page .sns-ui-send{", "flex:0 0 42px!important;width:42px!important;height:42px!important;", "}", "body.sns-chat-page .sns-ui-plus{", "background:transparent!important;color:#9b9ba5!important;", "font:400 22px/34px Arial,sans-serif!important;", "transition:background .15s ease,color .15s ease,transform .15s ease!important;", "}", "body.sns-chat-page .sns-ui-plus:hover{background:rgba(0,0,0,.05)!important;color:#74747e!important;}", "body.sns-chat-page .sns-ui-plus:active{transform:scale(.94)!important;}", "body.sns-chat-page .sns-ui-format{", "background:transparent!important;color:#85858f!important;", "font:700 12px/34px Arial,sans-serif!important;letter-spacing:-.35px!important;", "cursor:pointer!important;", "transition:background .15s ease,color .15s ease,transform .15s ease!important;", "}", "body.sns-chat-page .sns-ui-format:hover,body.sns-chat-page .sns-ui-format.is-active{", "background:rgba(0,0,0,.055)!important;color:var(--sns-own-g1,#b65f3a)!important;", "}", "body.sns-chat-page .sns-ui-format:active{transform:scale(.94)!important;}", "body.sns-chat-page .sns-ui-format:disabled{opacity:.3!important;cursor:default!important;}", "body.sns-chat-page .sns-ui-send{background:transparent!important;color:var(--sns-own-g1,#b65f3a)!important;font-size:19px!important;}", "body.sns-chat-page #sns-format-toolbar{", "position:absolute!important;left:102px!important;right:58px!important;bottom:64px!important;z-index:2600!important;", "display:none!important;align-items:center!important;justify-content:flex-start!important;gap:4px!important;", "max-width:none!important;", "padding:5px 6px!important;overflow-x:auto!important;overflow-y:hidden!important;", "background:rgba(255,255,255,.985)!important;", "border:1px solid rgba(0,0,0,.065)!important;border-radius:10px!important;", "box-shadow:0 7px 18px rgba(18,20,28,.09),0 18px 40px rgba(18,20,28,.12)!important;", "white-space:nowrap!important;scrollbar-width:none!important;", "}", "body.sns-chat-page #sns-format-toolbar::-webkit-scrollbar{display:none!important;}", "body.sns-chat-page #sns-format-toolbar.is-open{display:flex!important;}", "body.sns-chat-page #sns-format-toolbar .sns-format-action{", "flex:0 0 31px!important;width:auto!important;min-width:31px!important;height:31px!important;", "margin:0!important;padding:0 7px!important;", "display:inline-flex!important;align-items:center!important;justify-content:center!important;", "border:0!important;border-radius:7px!important;", "background:transparent!important;color:#4e4e55!important;", "font:700 11px/31px Arial,sans-serif!important;", "cursor:pointer!important;box-sizing:border-box!important;", "}", "body.sns-chat-page #sns-format-toolbar .sns-format-action:hover{", "background:rgba(0,0,0,.055)!important;color:var(--sns-own-g1,#b65f3a)!important;", "}", "body.sns-chat-page #sns-format-toolbar .sns-format-link{min-width:39px!important;font-size:9px!important;letter-spacing:.2px!important;}", "body.sns-chat-page #sns-format-toolbar .sns-format-divider{", "flex:0 0 1px!important;width:1px!important;height:19px!important;margin:0 2px!important;background:rgba(0,0,0,.09)!important;", "}", "body.sns-chat-page> #sns-attach-menu.sns-attach-menu-portal{", "position:fixed!important;", "z-index:2147482500!important;", "right:auto!important;", "bottom:auto!important;", "min-width:145px!important;", "pointer-events:auto!important;", "isolation:isolate!important;", "}", "body.sns-chat-page> #sns-attach-menu.sns-attach-menu-portal.is-open{", "display:block!important;", "visibility:visible!important;", "opacity:1!important;", "pointer-events:auto!important;", "}", "body.sns-chat-page #sns-composer-ui> #sns-attach-menu.sns-attach-menu-mobile-inline{", "position:absolute!important;", "z-index:2147482500!important;", "right:auto!important;", "top:auto!important;", "pointer-events:auto!important;", "isolation:isolate!important;", "}", "body.sns-chat-page #sns-composer-ui> #sns-attach-menu.sns-attach-menu-mobile-inline.is-open{", "display:block!important;", "visibility:visible!important;", "opacity:1!important;", "pointer-events:auto!important;", "}", "body.sns-chat-page> #sns-attach-menu.sns-attach-menu-portal .sns-attach-photo{", "position:relative!important;", "z-index:2!important;", "display:flex!important;", "align-items:center!important;", "gap:8px!important;", "white-space:nowrap!important;", "pointer-events:auto!important;", "cursor:pointer!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message .sns-controls{", "top:0!important;bottom:0!important;height:100%!important;width:26px!important;", "transform:none!important;", "z-index:80!important;opacity:.58!important;", "transition:opacity .15s ease!important;", "pointer-events:none!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message:hover .sns-controls{opacity:1!important;}", "body.sns-chat-page #pun-viewtopic .sns-own .sns-controls{left:-31px!important;}", "body.sns-chat-page #pun-viewtopic .sns-other .sns-controls{right:-31px!important;}", "body.sns-chat-page #pun-viewtopic .sns-controls>.sns-direct-reply{pointer-events:auto!important;}", "body.sns-chat-page #pun-viewtopic .sns-message .post-body,", "body.sns-chat-page #pun-viewtopic .sns-message .post-box,", "body.sns-chat-page #pun-viewtopic .sns-message>.container{overflow:visible!important;}", "body.sns-chat-page #pun-viewtopic .sns-menu-toggle{", "position:absolute!important;top:auto!important;bottom:-21px!important;", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:24px!important;height:20px!important;margin:0!important;padding:0!important;", "border:0!important;border-radius:0!important;", "background:transparent!important;", "color:rgba(255,255,255,.92)!important;", "box-shadow:none!important;", "-webkit-backdrop-filter:none!important;backdrop-filter:none!important;", "font:900 10px/10px Arial,sans-serif!important;letter-spacing:1.4px!important;", "opacity:.58!important;filter:drop-shadow(0 1px 2px rgba(20,20,20,.42))!important;", "cursor:pointer!important;box-sizing:border-box!important;", "z-index:95!important;", "transition:color .15s ease,transform .15s ease,opacity .15s ease!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-own .sns-menu-toggle{right:7px!important;left:auto!important;}", "body.sns-chat-page #pun-viewtopic .sns-other .sns-menu-toggle{left:7px!important;right:auto!important;}", "body.sns-chat-page #pun-viewtopic .sns-message:hover .sns-menu-toggle{opacity:1!important;}", "body.sns-chat-page #pun-viewtopic .sns-menu-toggle:hover{", "background:transparent!important;color:#fff!important;", "box-shadow:none!important;", "opacity:1!important;", "transform:scale(1.10)!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-menu-toggle:active{transform:scale(.96)!important;}", "body.sns-chat-page #pun-viewtopic .sns-message .sns-menu{", "top:auto!important;bottom:-18px!important;", "margin:0!important;z-index:10002!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-own .sns-menu{right:35px!important;left:auto!important;}", "body.sns-chat-page #pun-viewtopic .sns-other .sns-menu{left:35px!important;right:auto!important;}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-menu-active{z-index:10000!important;}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-menu-active .sns-controls{z-index:10001!important;opacity:1!important;}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-menu-active .sns-menu-toggle{", "background:transparent!important;color:#fff!important;", "box-shadow:none!important;", "transform:scale(1.08)!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-menu-active .sns-menu{z-index:10002!important;}", "@media (max-width:650px){", "body.sns-chat-page #pun-viewtopic .sns-message .sns-controls{opacity:.78!important;width:25px!important;}", "body.sns-chat-page #pun-viewtopic .sns-menu-toggle{bottom:-20px!important;width:23px!important;height:19px!important;font-size:10px!important;letter-spacing:1.3px!important;}", "body.sns-chat-page #pun-viewtopic .sns-own .sns-controls{left:-28px!important;}", "body.sns-chat-page #pun-viewtopic .sns-other .sns-controls{right:-28px!important;}", "body.sns-chat-page .sns-direct-reply{width:25px!important;height:25px!important;}", "body.sns-chat-page .sns-action-reply-icon,body.sns-chat-page .sns-action-reply-icon svg{width:17px!important;height:17px!important;}", "}", "#sns-settings-backdrop{", "position:fixed;inset:0;z-index:10000;display:none;align-items:center;justify-content:center;", "padding:22px;background:rgba(15,16,22,.55);backdrop-filter:blur(5px);", "}", "#sns-settings-backdrop.is-open{display:flex;}", "#sns-settings-modal{", "box-sizing:border-box;width:min(610px,100%);height:min(760px,88vh);max-height:88vh;overflow:hidden;", "display:flex;flex-direction:column;", "padding:24px;background:#f7f7f8;color:#25252b;border-radius:18px;", "box-shadow:0 25px 80px rgba(0,0,0,.30);font:12px/1.4 Arial,sans-serif;", "}", "#sns-settings-modal .sns-settings-head{display:flex;flex:0 0 auto;align-items:center;justify-content:space-between;margin-bottom:12px;}", "#sns-settings-modal .sns-settings-title{font-size:17px;font-weight:700;}", "#sns-settings-modal .sns-settings-close{width:32px;height:32px;border:0;border-radius:50%;cursor:pointer;background:#e8e8eb;color:#555;font-size:18px;}", "#sns-settings-modal .sns-settings-grid{", "display:grid;grid-template-columns:1fr 1fr;gap:14px;", "flex:1 1 auto;min-height:0;overflow-y:auto;overflow-x:hidden;", "align-content:start;padding:2px 7px 4px 0;", "scrollbar-gutter:stable;", "}", "#sns-settings-modal .sns-settings-grid::-webkit-scrollbar{width:7px;}", "#sns-settings-modal .sns-settings-grid::-webkit-scrollbar-thumb{background:rgba(70,70,78,.20);border-radius:20px;}", "#sns-settings-modal .sns-settings-grid::-webkit-scrollbar-track{background:transparent;}", "#sns-settings-modal .sns-field{display:flex;flex-direction:column;gap:6px;}", "#sns-settings-modal .sns-field.sns-span-2{grid-column:1/-1;}", "#sns-settings-modal label{font-size:10px;font-weight:700;letter-spacing:.5px;text-transform:uppercase;color:#777;}", "#sns-settings-modal input[type=text],#sns-settings-modal input[type=url]{", "box-sizing:border-box;width:100%;height:38px;padding:0 11px;border:1px solid #dddde2;border-radius:9px;background:#fff;color:#222;outline:none;", "}", "#sns-settings-modal input[type=color]{width:100%;height:38px;padding:3px;border:1px solid #dddde2;border-radius:9px;background:#fff;}", "#sns-settings-modal input[type=range]{width:100%;}", "#sns-settings-modal .sns-inline{display:flex;gap:8px;align-items:center;}", "#sns-settings-modal .sns-inline input{flex:1 1 auto;}", "#sns-settings-modal .sns-settings-preview-dock{", "flex:0 0 auto;margin:0 0 14px!important;padding:0!important;", "}", "#sns-settings-modal .sns-settings-preview-dock>label{", "display:block;margin-bottom:6px;", "}", "#sns-settings-modal .sns-settings-preview{", "position:relative;min-height:108px;overflow:hidden;border-radius:12px;background-size:cover;background-position:center;background-color:#7d9be9;", "box-shadow:0 8px 22px rgba(0,0,0,.10);", "}", '#sns-settings-modal .sns-settings-preview:before{content:"";position:absolute;inset:0;background:var(--preview-overlay,#8b5a3c);opacity:var(--preview-opacity,.36);z-index:1;}', '#sns-settings-modal .sns-preview-noise{position:absolute;inset:0;z-index:2;pointer-events:none;opacity:var(--preview-noise,0);mix-blend-mode:overlay;filter:contrast(175%);background-image:url("data:image/svg+xml,%3Csvg xmlns=%22http://www.w3.org/2000/svg%22 width=%22120%22 height=%22120%22%3E%3Cfilter id=%22n%22%3E%3CfeTurbulence type=%22fractalNoise%22 baseFrequency=%220.78%22 numOctaves=%224%22 stitchTiles=%22stitch%22/%3E%3C/filter%3E%3Crect width=%22100%25%22 height=%22100%25%22 filter=%22url(%23n)%22 opacity=%221%22/%3E%3C/svg%3E");background-size:120px 120px;}', "#sns-settings-modal .sns-preview-content{position:relative;z-index:3;padding:11px 14px;color:#fff;}", "#sns-settings-modal .sns-preview-head{display:flex;align-items:center;gap:10px;}", "#sns-settings-modal .sns-preview-avatar{", "position:relative;", "flex:0 0 38px;", "width:38px;", "height:38px;", "overflow:hidden;", "border-radius:50%;", "border:2px solid rgba(255,255,255,.82);", "background-color:rgba(255,255,255,.18);", "background-image:var(--preview-avatar-image,none);", "background-size:cover;", "background-position:center;", "filter:grayscale(var(--preview-avatar-gray,0%));", "isolation:isolate;", "}", "#sns-settings-modal .sns-preview-avatar:before{", 'content:"";position:absolute;inset:0;z-index:1;pointer-events:none;', "background:var(--preview-avatar-tint,#8a5a3a);", "opacity:var(--preview-avatar-tint-opacity,0);", "mix-blend-mode:var(--preview-avatar-blend,normal);", "}", "#sns-settings-modal .sns-preview-avatar:after{", 'content:"";position:absolute;inset:0;z-index:2;pointer-events:none;', "opacity:var(--preview-avatar-noise,0);mix-blend-mode:overlay;filter:contrast(165%);", 'background-image:url("data:image/svg+xml,%3Csvg xmlns=%22http://www.w3.org/2000/svg%22 width=%2270%22 height=%2270%22%3E%3Cfilter id=%22n%22%3E%3CfeTurbulence type=%22fractalNoise%22 baseFrequency=%220.8%22 numOctaves=%224%22 stitchTiles=%22stitch%22/%3E%3C/filter%3E%3Crect width=%22100%25%22 height=%22100%25%22 filter=%22url(%23n)%22 opacity=%220.85%22/%3E%3C/svg%3E");', "background-size:70px 70px;", "}", "#sns-settings-modal .sns-preview-dialog{position:relative;box-sizing:border-box;width:78%;min-height:54px;margin:8px 0 0 auto;padding:9px;border-radius:11px;overflow:hidden;box-shadow:0 8px 24px rgba(0,0,0,.14);background:#fbfaf7;}", '#sns-settings-modal .sns-preview-dialog:before{content:"";position:absolute;inset:-12px;background-color:var(--preview-dialog-color,#fbfaf7);background-image:var(--preview-dialog-image,none);background-size:var(--preview-dialog-size,cover);background-repeat:var(--preview-dialog-repeat,no-repeat);background-position:var(--preview-dialog-position,center);filter:blur(var(--preview-dialog-blur,0px));transform:scale(1.04);}', '#sns-settings-modal .sns-preview-dialog:after{content:"";position:absolute;inset:0;pointer-events:none;opacity:var(--preview-dialog-noise,0);mix-blend-mode:overlay;filter:contrast(175%);background-image:url("data:image/svg+xml,%3Csvg xmlns=%22http://www.w3.org/2000/svg%22 width=%22120%22 height=%22120%22%3E%3Cfilter id=%22n%22%3E%3CfeTurbulence type=%22fractalNoise%22 baseFrequency=%220.78%22 numOctaves=%224%22 stitchTiles=%22stitch%22/%3E%3C/filter%3E%3Crect width=%22100%25%22 height=%22100%25%22 filter=%22url(%23n)%22 opacity=%221%22/%3E%3C/svg%3E");background-size:120px 120px;}', "#sns-settings-modal .sns-preview-bubble{position:relative;z-index:2;display:inline-block;margin-top:10px;padding:7px 11px;border-radius:18px;background:linear-gradient(var(--preview-angle,135deg),var(--preview-g1,#b65f3a),var(--preview-g2,#d7a44a));color:#fff;}", "#sns-settings-modal select{box-sizing:border-box;width:100%;height:38px;padding:0 10px;border:1px solid #dddde2;border-radius:9px;background:#fff;color:#222;outline:none;}", "#sns-settings-modal .sns-settings-section{grid-column:1/-1;margin:5px 0 -3px;padding-top:4px;border-top:1px solid #e5e5e8;color:#a06c48;font-size:9px;font-weight:800;letter-spacing:1.2px;text-transform:uppercase;}", "#sns-settings-modal .sns-settings-grid>.sns-settings-section:first-child{margin-top:0;padding-top:0;border-top:0;}", "#sns-settings-modal .sns-dialog-option.is-hidden{display:none!important;}", "#sns-settings-modal .sns-range-note{font-size:9px;color:#999;margin-top:-2px;}", "#sns-settings-modal .sns-participants-box{display:flex;flex-direction:column;gap:8px;}", "#sns-settings-modal .sns-participant-settings-row{", "display:flex;align-items:center;gap:9px;min-height:38px;padding:6px 8px;", "border:1px solid #e1e1e5;border-radius:10px;background:#fff;", "}", "#sns-settings-modal .sns-participant-settings-avatar{", "display:flex;align-items:center;justify-content:center;flex:0 0 28px;width:28px;height:28px;", "object-fit:cover;border-radius:50%;background:#ececef;color:#666;font:700 10px Arial,sans-serif;", "}", "#sns-settings-modal .sns-participant-settings-name{flex:1 1 auto;min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;font-weight:700;}", "#sns-settings-modal .sns-participant-color{flex:0 0 34px!important;width:34px!important;height:28px!important;padding:2px!important;border-radius:8px!important;cursor:pointer;}", "#sns-settings-modal .sns-settings-tabs{display:flex;flex:0 0 auto;gap:6px;margin:0 0 12px;padding:4px;overflow-x:auto;background:#ececef;border-radius:11px;scrollbar-width:none;}", "#sns-settings-modal .sns-settings-tabs::-webkit-scrollbar{display:none;}", "#sns-settings-modal .sns-settings-tab{flex:1 0 auto;min-width:max-content;height:32px;padding:0 11px;border:0;border-radius:8px;cursor:pointer;background:transparent;color:#777;font:700 9px/32px Arial,sans-serif;letter-spacing:.45px;text-transform:uppercase;white-space:nowrap;}", "#sns-settings-modal .sns-settings-tab.is-active{background:#fff;color:#28282e;box-shadow:0 2px 8px rgba(0,0,0,.08);}", "#sns-settings-modal .sns-settings-grid{display:block!important;}", "#sns-settings-modal .sns-settings-pane{display:none;grid-template-columns:1fr 1fr;gap:14px;align-content:start;}", "#sns-settings-modal .sns-settings-pane.is-active{display:grid;}", "#sns-settings-modal .sns-settings-pane>.sns-settings-section:first-child{margin-top:0;padding-top:0;border-top:0;}", "body.sns-chat-page #pun-viewtopic .sns-author-name{font-size:10px!important;font-weight:800!important;letter-spacing:.18px!important;opacity:1!important;}", "body.sns-chat-page #pun-viewtopic .sns-author-name[data-sns-profile-link],", "body.sns-chat-page #pun-viewtopic .sns-mini-avatar[data-sns-profile-link],", "body.sns-chat-page .sns-participant-avatar[data-participant-id]{cursor:pointer!important;}", "body.sns-chat-page #pun-viewtopic .sns-author-name[data-sns-profile-link]:hover{text-decoration:underline;}", "body.sns-chat-page #pun-viewtopic .sns-mini-avatar[data-sns-profile-link],", "body.sns-chat-page .sns-participant-avatar[data-participant-id]{transition:opacity .15s ease,transform .15s ease;}", "body.sns-chat-page #pun-viewtopic .sns-mini-avatar[data-sns-profile-link]:hover,", "body.sns-chat-page .sns-participant-avatar[data-participant-id]:hover{opacity:.82;transform:scale(1.04);}", "#sns-settings-modal .sns-participant-owner-badge{font-size:8px;color:#999;text-transform:uppercase;letter-spacing:.5px;}", "#sns-settings-modal .sns-participant-remove{", "flex:0 0 26px;width:26px;height:26px;border:0;border-radius:50%;cursor:pointer;", "background:#eeeeF1;color:#777;font:16px/26px Arial,sans-serif;", "}", "#sns-settings-modal .sns-participant-add-row{display:flex;gap:8px;margin-top:2px;}", "#sns-settings-modal .sns-participant-add-row input{flex:1 1 auto;}", "#sns-settings-modal .sns-participant-add{", "flex:0 0 auto;height:38px;padding:0 12px;border:0;border-radius:9px;cursor:pointer;", "background:#27272d;color:#fff;font-weight:700;", "}", "#sns-settings-modal .sns-participant-readonly-note{margin-top:4px;color:#999;font-size:9px;}", "body.sns-chat-page #sns-access-note{", "position:relative;z-index:20;box-sizing:border-box;", "width:calc(100% - var(--sns-card-offset) + var(--sns-card-overhang));margin:0 0 0 var(--sns-card-offset);", "min-height:48px;padding:16px 18px;", "border-radius:0 0 16px 16px;background:rgba(242,242,244,.98);color:#8a8a8f;", "font:500 10px/16px Arial,sans-serif;letter-spacing:.08px;text-align:center;", "box-shadow:0 8px 18px rgba(20,18,16,.08),0 24px 52px rgba(20,22,30,.18),0 42px 88px rgba(20,22,30,.20);", "}", "body.sns-chat-page.sns-chat-readonly #sns-composer-ui{display:none!important;}", "body.sns-chat-page.sns-chat-readonly #sns-attach-menu{display:none!important;}", "body.sns-chat-page .sns-unauthorized-post{display:none!important;}", "body.sns-chat-page #pun-viewtopic .pagelink{display:none!important;}", "body.sns-chat-page #sns-pages-loading{", "position:absolute;left:50%;top:50%;z-index:30;transform:translate(-50%,-50%);", "padding:7px 10px;border-radius:20px;background:rgba(255,255,255,.78);", "color:#777;font:10px/1 Arial,sans-serif;backdrop-filter:blur(6px);", "pointer-events:none;", "}", "#sns-settings-modal .sns-settings-actions{", "display:flex;flex:0 0 auto;justify-content:flex-end;gap:9px;", "margin-top:12px;padding-top:12px;border-top:1px solid #e1e1e5;", "background:#f7f7f8;", "}", "#sns-settings-modal .sns-settings-actions button{height:38px;padding:0 16px;border:0;border-radius:10px;cursor:pointer;font-weight:700;}", "#sns-settings-cancel{background:#e7e7ea;color:#555;}", "#sns-settings-save{background:#25252b;color:#fff;}", "#sns-settings-status{margin-right:auto;align-self:center;color:#777;font-size:10px;}", "@media(max-width:650px){", "#sns-settings-backdrop{padding:10px;}", "#sns-settings-modal{width:100%;height:92vh;max-height:92vh;padding:16px;}", "#sns-settings-modal .sns-settings-grid{grid-template-columns:1fr;}", "#sns-settings-modal .sns-settings-pane{grid-template-columns:1fr;}", "#sns-settings-modal .sns-field.sns-span-2,#sns-settings-modal .sns-settings-section{grid-column:1;}", "#sns-settings-modal .sns-settings-preview{min-height:96px;}", "}", "body.sns-chat-page.sns-settings-upload-pending #image-area{visibility:hidden!important;}", "body.sns-chat-page> #image-area.sns-settings-upload-popover{", "position:fixed!important;", "left:var(--sns-upload-left,50vw)!important;", "top:var(--sns-upload-top,50vh)!important;", "right:auto!important;", "bottom:auto!important;", "transform:none!important;", "z-index:2147482500!important;", "display:block!important;", "visibility:visible!important;", "box-sizing:border-box!important;", "width:360px!important;", "max-width:calc(100vw - 24px)!important;", "max-height:260px!important;", "margin:0!important;", "padding:34px 12px 12px!important;", "overflow:auto!important;", "background:#f8f8fa!important;", "color:#222!important;", "border:1px solid rgba(0,0,0,.13)!important;", "border-radius:12px!important;", "box-shadow:0 18px 48px rgba(0,0,0,.28)!important;", "}", "body.sns-chat-page> #image-area.sns-settings-upload-popover:before{", 'content:"\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u0438\u0437\u043e\u0431\u0440\u0430\u0436\u0435\u043d\u0438\u044f";', "position:absolute;", "left:12px;", "top:11px;", "font:700 9px/1 Arial,sans-serif;", "letter-spacing:.8px;", "text-transform:uppercase;", "color:#777;", "}", "body.sns-chat-page> #image-area.sns-settings-upload-popover .sns-settings-popover-close{", "position:absolute!important;", "right:8px!important;", "top:7px!important;", "z-index:50!important;", "display:flex!important;", "align-items:center!important;", "justify-content:center!important;", "width:25px!important;", "height:25px!important;", "margin:0!important;", "padding:0!important;", "border:0!important;", "border-radius:50%!important;", "cursor:pointer!important;", "background:#e6e6e9!important;", "color:#555!important;", "font:20px/1 Arial,sans-serif!important;", "}", "body.sns-chat-page> #image-area.sns-settings-upload-popover .sns-settings-popover-close:hover{background:#d9d9dd!important;color:#222!important;}", "body.sns-chat-page> #image-area.sns-settings-upload-popover .sns-image-area-close{display:none!important;}", "@media(max-width:650px){", "body.sns-chat-page> #image-area.sns-settings-upload-popover{", "left:12px!important;", "right:12px!important;", "top:50%!important;", "width:auto!important;", "max-width:none!important;", "transform:translateY(-50%)!important;", "}", "}", "body.sns-chat-page.sns-phone-device #sns-chat-shell>.topic{", "border-radius:16px 16px 0 0!important;", "overflow-x:hidden!important;", "overflow-y:auto!important;", "clip-path:inset(0 round 16px 16px 0 0)!important;", "-webkit-mask-image:-webkit-radial-gradient(white,black)!important;", "}", "body.sns-chat-page.sns-phone-device #sns-chat-shell>.sns-dialog-bg-layer{", "border-radius:16px 16px 0 0!important;", "overflow:hidden!important;", "clip-path:inset(0 round 16px 16px 0 0)!important;", "-webkit-mask-image:-webkit-radial-gradient(white,black)!important;", "}", "body.sns-chat-page.sns-phone-device #sns-chat-shell>.sns-dialog-bg-layer>.sns-dialog-bg-image{", "border-radius:inherit!important;", "}", "@media(max-width:650px){", "body.sns-chat-page #sns-chat-shell{--sns-geo-offset:18px;--sns-card-offset:24px;--sns-card-overhang:0px;--sns-geo-bottom:20px;--sns-bg-height:520px;padding-bottom:20px!important;}", "body.sns-chat-page #sns-chat-header{min-height:132px!important;padding:22px 14px 42px 18px!important;}", "body.sns-chat-page #sns-chat-header .sns-head-title{font-size:17px!important;}", "body.sns-chat-page .sns-head-participants{max-width:min(190px,48vw);gap:5px;}", "body.sns-chat-page .sns-participant-avatar{flex-basis:31px;width:31px;height:31px;}", "body.sns-chat-page .sns-settings-open,body.sns-chat-page .sns-search-open{flex-basis:31px;width:31px;height:31px;}", "body.sns-chat-page .sns-settings-icon{width:17px!important;height:17px!important;}", "body.sns-chat-page .sns-search-open svg{width:16px!important;height:16px!important;}", "body.sns-chat-page #sns-chat-shell>.topic{width:calc(100% - 24px)!important;height:545px!important;margin:-40px 0 0 24px!important;padding:36px 13px 24px!important;scroll-padding-top:14px!important;border-radius:12px 12px 0 0!important;box-shadow:0 -7px 16px rgba(20,22,30,.11),0 8px 20px rgba(20,22,30,.14),0 24px 50px rgba(20,22,30,.18)!important;}", "body.sns-chat-page #sns-chat-shell>.sns-dialog-bg-layer{border-radius:12px 12px 0 0!important;box-shadow:0 -9px 20px rgba(20,22,30,.11),0 18px 42px rgba(20,22,30,.13)!important;}", "body.sns-chat-page #sns-composer-ui{width:calc(100% - 24px)!important;margin:0 0 0 24px!important;border-radius:0 0 12px 12px!important;box-shadow:0 18px 42px rgba(20,22,30,.17),inset 0 1px 0 rgba(255,255,255,.68)!important;grid-template-columns:30px 30px minmax(0,1fr) 38px!important;column-gap:6px!important;}", "body.sns-chat-page .sns-ui-plus,body.sns-chat-page .sns-ui-format{width:30px!important;height:30px!important;flex-basis:30px!important;}", "body.sns-chat-page .sns-ui-plus{font:400 20px/30px Arial,sans-serif!important;}", "body.sns-chat-page .sns-ui-format{font:700 10px/30px Arial,sans-serif!important;}", "body.sns-chat-page #sns-format-toolbar{left:78px!important;right:46px!important;bottom:59px!important;max-width:none!important;}", "body.sns-chat-page #sns-access-note{width:calc(100% - 24px)!important;margin-left:24px!important;border-radius:0 0 12px 12px!important;}", "body.sns-chat-page.sns-chat-readonly #sns-composer-ui{display:none!important;}", "#sns-settings-modal .sns-settings-grid{grid-template-columns:1fr;}", "#sns-settings-modal .sns-field.sns-span-2{grid-column:auto;}", "}" ].join("");
        document.head.appendChild(style);
      }
      function applyConfig(config) {
        currentConfig = $.extend({}, DEFAULTS, config || {});
        var $shell = $("#sns-chat-shell");
        var $header = $("#sns-chat-header");
        var $topic = getTopic();
        if (!$shell.length || !$header.length) return;
        var original = originalTopicTitle();
        $header.find(".sns-head-title").attr("data-sns-original-title", original).text(currentConfig.title || original);
        var ownerAvatar = getPostAvatar(getFirstPost());
        var avatar = currentConfig.chatAvatar || ownerAvatar;
        var $headAvatar = $header.find(".sns-head-avatar").first();
        if (avatar) {
          if (!$headAvatar.is("img")) {
            $headAvatar.replaceWith('<img class="sns-head-avatar" alt="">');
            $headAvatar = $header.find(".sns-head-avatar").first();
          }
          $headAvatar.attr("src", avatar);
        }
        ensureNoiseLayer();
        ensureDialogBackgroundLayer();
        positionDialogBackgroundLayer();
        var shellNode = $shell.get(0);
        if (shellNode && shellNode.style) {
          shellNode.style.setProperty("--sns-overlay-color", safeColor(currentConfig.overlayColor, DEFAULTS.overlayColor));
          shellNode.style.setProperty("--sns-overlay-opacity", String(clamp(currentConfig.overlayOpacity, 0, 90) / 100));
          shellNode.style.setProperty("--sns-noise-opacity", String(clamp(currentConfig.noiseOpacity, 0, 100) / 100));
          shellNode.style.setProperty("--sns-avatar-gray", String(clamp(currentConfig.avatarGrayscale, 0, 100)) + "%");
          shellNode.style.setProperty("--sns-avatar-noise-opacity", String(clamp(currentConfig.avatarNoiseOpacity, 0, 100) / 100));
          shellNode.style.setProperty("--sns-avatar-tint", safeColor(currentConfig.avatarTint, DEFAULTS.avatarTint));
          shellNode.style.setProperty("--sns-avatar-tint-opacity", String(clamp(currentConfig.avatarTintOpacity, 0, 80) / 100));
          shellNode.style.setProperty("--sns-avatar-blend", safeChoice(currentConfig.avatarBlendMode, [ "normal", "multiply", "soft-light", "color" ], DEFAULTS.avatarBlendMode));
          shellNode.style.setProperty("--sns-own-g1", safeColor(currentConfig.ownGradient1, DEFAULTS.ownGradient1));
          shellNode.style.setProperty("--sns-own-g2", safeColor(currentConfig.ownGradient2, DEFAULTS.ownGradient2));
          shellNode.style.setProperty("--sns-gradient-angle", String(clamp(currentConfig.gradientAngle, 0, 360)) + "deg");
          shellNode.style.setProperty("--sns-other-bubble", safeColor(currentConfig.otherBubble, DEFAULTS.otherBubble));
          shellNode.style.setProperty("--sns-other-text", readableTextColor(currentConfig.otherBubble));
          if (currentConfig.backgroundUrl) {
            shellNode.style.setProperty("--sns-bg-image", escapedCssUrl(currentConfig.backgroundUrl));
          } else {
            shellNode.style.setProperty("--sns-bg-image", "none");
          }
          var dialogBg = dialogBackgroundCss(currentConfig);
          shellNode.style.setProperty("--sns-dialog-color", dialogBg.color);
          shellNode.style.setProperty("--sns-dialog-image", dialogBg.image);
          shellNode.style.setProperty("--sns-dialog-size", dialogBg.size);
          shellNode.style.setProperty("--sns-dialog-repeat", dialogBg.repeat);
          shellNode.style.setProperty("--sns-dialog-position", dialogBg.position);
          shellNode.style.setProperty("--sns-dialog-noise-opacity", String(clamp(currentConfig.dialogNoiseOpacity, 0, 100) / 100));
          shellNode.style.setProperty("--sns-dialog-blur", String(clamp(currentConfig.dialogBlur, 0, 18)) + "px");
          shellNode.style.backgroundImage = "none";
        }
        renderHeaderTools();
        applyParticipantNameColors(currentConfig);
      }
      function ensureAccessNote() {
        var $note = $("#sns-access-note");
        if (!$note.length) {
          $note = $('<div id="sns-access-note">' + "\u0432\u044b \u043d\u0435 \u044f\u0432\u043b\u044f\u0435\u0442\u0435\u0441\u044c \u0443\u0447\u0430\u0441\u0442\u043d\u0438\u043a\u043e\u043c \u044d\u0442\u043e\u0433\u043e \u0447\u0430\u0442\u0430" + "</div>");
          var $topic = getTopic();
          if ($topic.length) {
            $topic.after($note);
          }
        }
        return $note;
      }
      function applyAccessControl() {
        var allowed = isCurrentParticipant(currentConfig);
        var $composer = $("#sns-composer-ui");
        var $note = ensureAccessNote();
        $composer.toggle(allowed);
        if (!allowed) {
          $composer.attr("aria-hidden", "true");
        } else {
          $composer.removeAttr("aria-hidden");
        }
        $note.toggle(!allowed);
        $("#sns-ui-input, #main-reply").prop("disabled", !allowed);
        $(".sns-ui-send, .sns-ui-plus, .sns-ui-format").prop("disabled", !allowed);
        if (!allowed) {
          $("#sns-attach-menu").removeClass("is-open").hide();
          $("#sns-format-toolbar").removeClass("is-open").attr("aria-hidden", "true");
          $(".sns-ui-format").removeClass("is-active").attr("aria-expanded", "false");
          $(".sns-settings-open").remove();
        }
        getTopic().find(".sns-message").each(function() {
          var $post = $(this);
          var author = getPostAuthor($post);
          $post.toggleClass("sns-unauthorized-post", !isParticipantIdentity(author, getPostUserId($post), currentConfig));
        });
        $("body").toggleClass("sns-chat-readonly", !allowed);
      }
      function renderHeaderTools() {
        var $header = $("#sns-chat-header");
        if (!$header.length) {
          return;
        }
        $header.find(".sns-head-tools").remove();
        var participants = effectiveParticipants(currentConfig);
        var $tools = $('<div class="sns-head-tools"></div>');
        var $participants = $('<div class="sns-head-participants"></div>');
        participants.forEach(function(participant) {
          participant = normalizeParticipant(participant);
          var avatar = knownAvatarForParticipant(participant);
          var displayName = participant.name || (participant.id ? "#" + participant.id : "?");
          var $node;
          if (avatar) {
            $node = $('<img class="sns-participant-avatar" alt="">').attr("src", avatar);
          } else {
            $node = $('<span class="sns-participant-avatar sns-participant-fallback"></span>').text((displayName.charAt(0) || "?").toUpperCase());
          }
          $node.attr("title", displayName).attr("data-participant-id", participant.id || "").attr("data-participant-name", participant.name || "").attr("role", participant.id ? "link" : "").attr("tabindex", participant.id ? "0" : "-1").appendTo($participants);
          if (!avatar) {
            resolveParticipantProfile(participant, function(data) {
              if (!data || data.found === false) {
                return;
              }
              var selectorId = String(data.id || participant.id || "");
              var $current = $participants.children("[data-participant-id]," + "[data-participant-name]").filter(function() {
                var rowId = String($(this).attr("data-participant-id") || "");
                if (selectorId && rowId) {
                  return selectorId === rowId;
                }
                return participantKey($(this).attr("data-participant-name")) === participantKey(participant.name);
              }).first();
              if (!$current.length) {
                return;
              }
              var name = cleanText(data.name || participant.name);
              if (data.avatar) {
                $current.replaceWith($('<img class="sns-participant-avatar" alt="">').attr("src", data.avatar).attr("title", name).attr("data-participant-id", data.id || participant.id || "").attr("data-participant-name", name).attr("role", "link").attr("tabindex", "0"));
              } else {
                $current.attr("title", name).attr("data-participant-id", data.id || participant.id || "").attr("data-participant-name", name);
              }
            });
          }
        });
        $tools.append($participants);
        $('<button type="button" class="sns-search-open" ' + 'title="\u041f\u043e\u0438\u0441\u043a \u043f\u043e \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f\u043c" ' + 'aria-label="\u041f\u043e\u0438\u0441\u043a \u043f\u043e \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f\u043c">' + '<svg viewBox="0 0 24 24" aria-hidden="true" focusable="false">' + '<circle cx="11" cy="11" r="6.5"></circle>' + '<path d="M16 16l4.2 4.2"></path>' + "</svg>" + "</button>").appendTo($tools);
        if (canEditAppearance(currentConfig)) {
          $('<button type="button" class="sns-settings-open" ' + 'title="\u041d\u0430\u0441\u0442\u0440\u043e\u0439\u043a\u0438 \u0447\u0430\u0442\u0430" ' + 'aria-label="\u041d\u0430\u0441\u0442\u0440\u043e\u0439\u043a\u0438 \u0447\u0430\u0442\u0430">' + '<svg class="sns-settings-icon" viewBox="0 0 24 24" aria-hidden="true" focusable="false">' + '<path d="M12.22 2h-.44a2 2 0 0 0-2 2v.18a2 2 0 0 1-1 1.73l-.43.25a2 2 0 0 1-2 0l-.15-.08a2 2 0 0 0-2.73.73l-.22.38a2 2 0 0 0 .73 2.73l.15.09a2 2 0 0 1 1 1.74v.5a2 2 0 0 1-1 1.74l-.15.09a2 2 0 0 0-.73 2.73l.22.38a2 2 0 0 0 2.73.73l.15-.08a2 2 0 0 1 2 0l.43.25a2 2 0 0 1 1 1.73V20a2 2 0 0 0 2 2h.44a2 2 0 0 0 2-2v-.18a2 2 0 0 1 1-1.73l.43-.25a2 2 0 0 1 2 0l.15.08a2 2 0 0 0 2.73-.73l.22-.38a2 2 0 0 0-.73-2.73l-.15-.09a2 2 0 0 1-1-1.74v-.5a2 2 0 0 1 1-1.74l.15-.09a2 2 0 0 0 .73-2.73l-.22-.38a2 2 0 0 0-2.73-.73l-.15.08a2 2 0 0 1-2 0l-.43-.25a2 2 0 0 1-1-1.73V4a2 2 0 0 0-2-2z"/>' + '<circle cx="12" cy="12" r="3"/>' + "</svg>" + "</button>").appendTo($tools);
        }
        $header.append($tools);
        applyAccessControl();
      }
      function buildSettingsModal() {
        if ($("#sns-settings-backdrop").length) return;
        var html = '<div id="sns-settings-backdrop">' + '<div id="sns-settings-modal">' + '<div class="sns-settings-head">' + '<div class="sns-settings-title">\u041d\u0430\u0441\u0442\u0440\u043e\u0439\u043a\u0438 \u0447\u0430\u0442\u0430</div>' + '<button type="button" class="sns-settings-close">\xd7</button>' + "</div>" + '<div class="sns-settings-grid">' + '<div class="sns-settings-section">\u0423\u0447\u0430\u0441\u0442\u043d\u0438\u043a\u0438 \u0447\u0430\u0442\u0430</div>' + '<div class="sns-field sns-span-2">' + "<label>\u0423\u0447\u0430\u0441\u0442\u043d\u0438\u043a\u0438</label>" + '<div id="sns-participants-editor" class="sns-participants-box"></div>' + '<div class="sns-participant-add-row" id="sns-participant-add-row">' + '<input type="text" id="sns-participant-name" maxlength="255" placeholder="\u0421\u0441\u044b\u043b\u043a\u0430 \u043d\u0430 \u043f\u0440\u043e\u0444\u0438\u043b\u044c: profile.php?id=...">' + '<button type="button" class="sns-participant-add">+ \u0414\u043e\u0431\u0430\u0432\u0438\u0442\u044c</button>' + "</div>" + '<span class="sns-range-note" id="sns-participants-note"></span>' + "</div>" + '<div class="sns-field sns-span-2">' + "<label>\u041d\u0430\u0437\u0432\u0430\u043d\u0438\u0435 \u0447\u0430\u0442\u0430</label>" + '<input type="text" id="sns-set-title" maxlength="80" placeholder="\u041d\u0430\u043f\u0440\u0438\u043c\u0435\u0440: saturday night">' + "</div>" + '<div class="sns-settings-section">\u0410\u0432\u0430\u0442\u0430\u0440 \u0447\u0430\u0442\u0430</div>' + '<div class="sns-field sns-span-2">' + "<label>\u0410\u0432\u0430\u0442\u0430\u0440</label>" + '<input type="url" id="sns-set-avatar" placeholder="\u041f\u0443\u0441\u0442\u043e = \u0430\u0432\u0430\u0442\u0430\u0440 \u0441\u043e\u0437\u0434\u0430\u0442\u0435\u043b\u044f \u0438\u0437 \u043f\u0440\u043e\u0444\u0438\u043b\u044f">' + '<span class="sns-range-note">\u0418\u043b\u0438 \u0432\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u043f\u0440\u044f\u043c\u0443\u044e \u0441\u0441\u044b\u043b\u043a\u0443 \u043d\u0430 \u0430\u0432\u0430\u0442\u0430\u0440</span>' + "</div>" + '<div class="sns-field">' + '<label>\u0427/\u0411: <span id="sns-avatar-gray-value"></span>%</label>' + '<input type="range" id="sns-set-avatar-gray" min="0" max="100" step="1">' + "</div>" + '<div class="sns-field">' + '<label>\u0428\u0443\u043c: <span id="sns-avatar-noise-value"></span>%</label>' + '<input type="range" id="sns-set-avatar-noise" min="0" max="100" step="1">' + "</div>" + '<div class="sns-field">' + "<label>\u0417\u0430\u043b\u0438\u0432\u043a\u0430 \u043f\u043e\u0432\u0435\u0440\u0445 \u0430\u0432\u0430\u0442\u0430\u0440\u0430</label>" + '<input type="color" id="sns-set-avatar-tint">' + "</div>" + '<div class="sns-field">' + '<label>\u041f\u0440\u043e\u0437\u0440\u0430\u0447\u043d\u043e\u0441\u0442\u044c \u0437\u0430\u043b\u0438\u0432\u043a\u0438: <span id="sns-avatar-tint-opacity-value"></span>%</label>' + '<input type="range" id="sns-set-avatar-tint-opacity" min="0" max="80" step="1">' + "</div>" + '<div class="sns-field sns-span-2">' + "<label>\u0420\u0435\u0436\u0438\u043c \u043d\u0430\u043b\u043e\u0436\u0435\u043d\u0438\u044f</label>" + '<select id="sns-set-avatar-blend">' + '<option value="normal">\u041e\u0431\u044b\u0447\u043d\u044b\u0439</option>' + '<option value="multiply">Multiply</option>' + '<option value="soft-light">Soft Light</option>' + '<option value="color">Color</option>' + "</select>" + "</div>" + '<div class="sns-settings-section">\u0412\u043d\u0435\u0448\u043d\u0438\u0439 \u0444\u043e\u043d</div>' + '<div class="sns-field sns-span-2">' + "<label>\u0424\u043e\u043d</label>" + '<input type="url" id="sns-set-bg" placeholder="https://...">' + '<span class="sns-range-note">\u0412\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u043f\u0440\u044f\u043c\u0443\u044e \u0441\u0441\u044b\u043b\u043a\u0443 \u043d\u0430 \u0438\u0437\u043e\u0431\u0440\u0430\u0436\u0435\u043d\u0438\u0435</span>' + "</div>" + '<div class="sns-field">' + "<label>\u041e\u0442\u0442\u0435\u043d\u043e\u043a \u043f\u043e\u0432\u0435\u0440\u0445 \u0444\u043e\u043d\u0430</label>" + '<input type="color" id="sns-set-overlay">' + "</div>" + '<div class="sns-field">' + '<label>\u041f\u0440\u043e\u0437\u0440\u0430\u0447\u043d\u043e\u0441\u0442\u044c \u043e\u0442\u0442\u0435\u043d\u043a\u0430: <span id="sns-overlay-value"></span>%</label>' + '<input type="range" id="sns-set-opacity" min="0" max="90" step="1">' + "</div>" + '<div class="sns-field sns-span-2">' + '<label>\u0428\u0443\u043c \u043f\u043e\u0432\u0435\u0440\u0445 \u0444\u043e\u043d\u0430: <span id="sns-noise-value"></span>%</label>' + '<input type="range" id="sns-set-noise" min="0" max="100" step="1">' + '<span class="sns-range-note">0% = \u0431\u0435\u0437 \u0437\u0435\u0440\u043d\u0430</span>' + "</div>" + '<div class="sns-settings-section">\u0421\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f</div>' + '<div class="sns-field">' + "<label>\u0413\u0440\u0430\u0434\u0438\u0435\u043d\u0442 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f \u2014 \u0446\u0432\u0435\u0442 1</label>" + '<input type="color" id="sns-set-g1">' + "</div>" + '<div class="sns-field">' + "<label>\u0413\u0440\u0430\u0434\u0438\u0435\u043d\u0442 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f \u2014 \u0446\u0432\u0435\u0442 2</label>" + '<input type="color" id="sns-set-g2">' + "</div>" + '<div class="sns-field">' + '<label>\u0423\u0433\u043e\u043b \u0433\u0440\u0430\u0434\u0438\u0435\u043d\u0442\u0430: <span id="sns-angle-value"></span>\xb0</label>' + '<input type="range" id="sns-set-angle" min="0" max="360" step="1">' + "</div>" + '<div class="sns-field">' + "<label>\u0427\u0443\u0436\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f</label>" + '<input type="color" id="sns-set-other">' + "</div>" + '<div class="sns-settings-section">\u0424\u043e\u043d \u0434\u0438\u0430\u043b\u043e\u0433\u0430</div>' + '<div class="sns-field sns-span-2">' + "<label>\u0422\u0438\u043f \u0444\u043e\u043d\u0430 \u0434\u0438\u0430\u043b\u043e\u0433\u0430</label>" + '<select id="sns-set-dialog-mode">' + '<option value="solid">\u041e\u0434\u043d\u043e\u0442\u043e\u043d\u043d\u044b\u0439</option>' + '<option value="gradient">\u0413\u0440\u0430\u0434\u0438\u0435\u043d\u0442</option>' + '<option value="photo">\u0424\u043e\u0442\u043e</option>' + '<option value="pattern">\u0423\u0437\u043e\u0440</option>' + "</select>" + "</div>" + '<div class="sns-field">' + '<label>\u0428\u0443\u043c \u0444\u043e\u043d\u0430 \u0434\u0438\u0430\u043b\u043e\u0433\u0430: <span id="sns-dialog-noise-value"></span>%</label>' + '<input type="range" id="sns-set-dialog-noise" min="0" max="100" step="1">' + "</div>" + '<div class="sns-field">' + '<label>\u0411\u043b\u044e\u0440 \u0444\u043e\u043d\u0430 \u0434\u0438\u0430\u043b\u043e\u0433\u0430: <span id="sns-dialog-blur-value"></span> px</label>' + '<input type="range" id="sns-set-dialog-blur" min="0" max="18" step="1">' + '<span class="sns-range-note">\u0411\u043b\u044e\u0440\u0438\u0442\u0441\u044f \u0442\u043e\u043b\u044c\u043a\u043e \u0444\u043e\u043d; \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f \u043e\u0441\u0442\u0430\u044e\u0442\u0441\u044f \u0440\u0435\u0437\u043a\u0438\u043c\u0438</span>' + "</div>" + '<div class="sns-field sns-dialog-option" data-dialog-mode="solid">' + "<label>\u0426\u0432\u0435\u0442 \u0434\u0438\u0430\u043b\u043e\u0433\u0430</label>" + '<input type="color" id="sns-set-dialog-color">' + "</div>" + '<div class="sns-field sns-dialog-option" data-dialog-mode="gradient">' + "<label>\u0413\u0440\u0430\u0434\u0438\u0435\u043d\u0442 \u2014 \u0446\u0432\u0435\u0442 1</label>" + '<input type="color" id="sns-set-dialog-g1">' + "</div>" + '<div class="sns-field sns-dialog-option" data-dialog-mode="gradient">' + "<label>\u0413\u0440\u0430\u0434\u0438\u0435\u043d\u0442 \u2014 \u0446\u0432\u0435\u0442 2</label>" + '<input type="color" id="sns-set-dialog-g2">' + "</div>" + '<div class="sns-field sns-span-2 sns-dialog-option" data-dialog-mode="gradient">' + '<label>\u0423\u0433\u043e\u043b \u0444\u043e\u043d\u043e\u0432\u043e\u0433\u043e \u0433\u0440\u0430\u0434\u0438\u0435\u043d\u0442\u0430: <span id="sns-dialog-angle-value"></span>\xb0</label>' + '<input type="range" id="sns-set-dialog-angle" min="0" max="360" step="1">' + "</div>" + '<div class="sns-field sns-span-2 sns-dialog-option" data-dialog-mode="photo">' + "<label>\u0424\u043e\u0442\u043e \u0434\u043b\u044f \u0444\u043e\u043d\u0430 \u0434\u0438\u0430\u043b\u043e\u0433\u0430</label>" + '<input type="url" id="sns-set-dialog-photo" placeholder="https://...">' + '<span class="sns-range-note">\u0412\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u043f\u0440\u044f\u043c\u0443\u044e \u0441\u0441\u044b\u043b\u043a\u0443 \u043d\u0430 \u0438\u0437\u043e\u0431\u0440\u0430\u0436\u0435\u043d\u0438\u0435</span>' + "</div>" + '<div class="sns-field sns-dialog-option" data-dialog-mode="photo">' + "<label>\u0417\u0430\u043b\u0438\u0432\u043a\u0430 \u043f\u043e\u0432\u0435\u0440\u0445 \u0444\u043e\u0442\u043e</label>" + '<input type="color" id="sns-set-dialog-photo-tint">' + "</div>" + '<div class="sns-field sns-dialog-option" data-dialog-mode="photo">' + '<label>\u041f\u0440\u043e\u0437\u0440\u0430\u0447\u043d\u043e\u0441\u0442\u044c \u0437\u0430\u043b\u0438\u0432\u043a\u0438: <span id="sns-dialog-photo-opacity-value"></span>%</label>' + '<input type="range" id="sns-set-dialog-photo-opacity" min="0" max="80" step="1">' + "</div>" + '<div class="sns-field sns-dialog-option" data-dialog-mode="pattern">' + "<label>\u0423\u0437\u043e\u0440</label>" + '<select id="sns-set-dialog-pattern">' + '<option value="dots">\u0422\u043e\u0447\u043a\u0438</option>' + '<option value="grid">\u0421\u0435\u0442\u043a\u0430</option>' + '<option value="diagonal">\u0414\u0438\u0430\u0433\u043e\u043d\u0430\u043b\u0438</option>' + '<option value="custom">\u0421\u0432\u043e\u0439 \u0443\u0437\u043e\u0440</option>' + "</select>" + "</div>" + '<div class="sns-field sns-dialog-option" data-dialog-mode="pattern">' + "<label>\u0424\u043e\u043d \u0443\u0437\u043e\u0440\u0430</label>" + '<input type="color" id="sns-set-dialog-pattern-base">' + "</div>" + '<div class="sns-field sns-dialog-option sns-pattern-built-in" data-dialog-mode="pattern">' + "<label>\u0426\u0432\u0435\u0442 \u0443\u0437\u043e\u0440\u0430</label>" + '<input type="color" id="sns-set-dialog-pattern-color">' + "</div>" + '<div class="sns-field sns-dialog-option sns-pattern-built-in" data-dialog-mode="pattern">' + '<label>\u0420\u0430\u0437\u043c\u0435\u0440 \u0443\u0437\u043e\u0440\u0430: <span id="sns-dialog-pattern-size-value"></span> px</label>' + '<input type="range" id="sns-set-dialog-pattern-size" min="10" max="160" step="1">' + "</div>" + '<div class="sns-field sns-span-2 sns-dialog-option sns-pattern-custom" data-dialog-mode="pattern">' + "<label>\u0421\u0432\u043e\u0439 \u043f\u0430\u0442\u0442\u0435\u0440\u043d</label>" + '<input type="url" id="sns-set-dialog-pattern-url" placeholder="https://...">' + '<span class="sns-range-note">\u0412\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u043f\u0440\u044f\u043c\u0443\u044e \u0441\u0441\u044b\u043b\u043a\u0443 \u043d\u0430 \u0443\u0437\u043e\u0440</span>' + "</div>" + '<div class="sns-field sns-span-2">' + "<label>\u041f\u0440\u0435\u0434\u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440</label>" + '<div class="sns-settings-preview" id="sns-settings-preview">' + '<div class="sns-preview-noise"></div>' + '<div class="sns-preview-content">' + '<div class="sns-preview-head">' + '<div class="sns-preview-avatar" id="sns-preview-avatar"></div>' + '<strong id="sns-preview-title">Chat</strong>' + "</div>" + '<div class="sns-preview-dialog">' + '<span class="sns-preview-bubble">\u0422\u0432\u043e\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435</span>' + "</div>" + "</div>" + "</div>" + "</div>" + "</div>" + '<div class="sns-settings-actions">' + '<span id="sns-settings-status"></span>' + '<button type="button" id="sns-settings-cancel">\u041e\u0442\u043c\u0435\u043d\u0430</button>' + '<button type="button" id="sns-settings-save">\u0421\u043e\u0445\u0440\u0430\u043d\u0438\u0442\u044c</button>' + "</div>" + "</div>" + "</div>";
        $("body").append(html);
        var $modal = $("#sns-settings-modal");
        var $previewField = $("#sns-settings-preview").closest(".sns-field");
        if ($modal.length && $previewField.length) {
          $previewField.removeClass("sns-span-2").addClass("sns-settings-preview-dock").detach();
          $modal.find(".sns-settings-head").after($previewField);
        }
        organizeSettingsTabs();
      }
      function organizeSettingsTabs() {
        var $modal = $("#sns-settings-modal");
        var $grid = $modal.find(".sns-settings-grid");
        if (!$modal.length || !$grid.length || $modal.find(".sns-settings-tabs").length) {
          return;
        }
        var tabs = [ {
          key: "participants",
          label: "\u0423\u0447\u0430\u0441\u0442\u043d\u0438\u043a\u0438"
        }, {
          key: "basic",
          label: "\u041e\u0441\u043d\u043e\u0432\u043d\u043e\u0435"
        }, {
          key: "messages",
          label: "\u0421\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f"
        }, {
          key: "dialog",
          label: "\u0424\u043e\u043d \u0434\u0438\u0430\u043b\u043e\u0433\u0430"
        } ];
        var $bar = $('<div class="sns-settings-tabs"></div>');
        var panes = {};
        tabs.forEach(function(tab, index) {
          $('<button type="button" class="sns-settings-tab"></button>').attr("data-settings-tab", tab.key).toggleClass("is-active", index === 0).text(tab.label).appendTo($bar);
          panes[tab.key] = $('<div class="sns-settings-pane"></div>').attr("data-settings-pane", tab.key).toggleClass("is-active", index === 0);
        });
        var activeKey = "participants";
        $grid.children().toArray().forEach(function(node) {
          var $node = $(node);
          if ($node.find("#sns-set-title").length) {
            activeKey = "basic";
            panes.basic.append($node);
            return;
          }
          if ($node.hasClass("sns-settings-section")) {
            var heading = cleanText($node.text()).toLowerCase();
            if (heading.indexOf("\u0443\u0447\u0430\u0441\u0442") !== -1) {
              activeKey = "participants";
            } else if (heading.indexOf("\u0430\u0432\u0430\u0442\u0430\u0440") !== -1 || heading.indexOf("\u0432\u043d\u0435\u0448\u043d") !== -1) {
              activeKey = "basic";
            } else if (heading.indexOf("\u0441\u043e\u043e\u0431\u0449") !== -1) {
              activeKey = "messages";
            } else if (heading.indexOf("\u0444\u043e\u043d \u0434\u0438\u0430") !== -1) {
              activeKey = "dialog";
            }
          }
          panes[activeKey].append($node);
        });
        $grid.empty();
        tabs.forEach(function(tab) {
          $grid.append(panes[tab.key]);
        });
        $grid.before($bar);
      }
      function participantEntriesFromEditor() {
        var result = [];
        var seen = {};
        $("#sns-participants-editor").children(".sns-participant-settings-row").each(function() {
          if ($(this).attr("data-owner") === "1") {
            return;
          }
          var item = {
            id: /^\d+$/.test(String($(this).attr("data-id") || "")) ? String($(this).attr("data-id")) : "",
            name: cleanText($(this).attr("data-name"))
          };
          var key = participantIdentityKey(item);
          if (!key || seen[key]) {
            return;
          }
          seen[key] = true;
          result.push(item);
        });
        return result;
      }
      function createParticipantSettingsRow(participant, ownerRow) {
        participant = normalizeParticipant(participant);
        var known = knownParticipantFromPosts(participant);
        if (known) {
          if (!participant.id && known.id) {
            participant.id = known.id;
          }
          if (known.name) {
            participant.name = known.name;
          }
        }
        var avatar = known && known.avatar ? known.avatar : knownAvatarForParticipant(participant);
        var displayName = participant.name || (participant.id ? "#" + participant.id : "?");
        var $row = $('<div class="sns-participant-settings-row"></div>').attr("data-id", participant.id || "").attr("title", participant.id ? "RusFF ID: " + participant.id : "").attr("data-name", participant.name || "").attr("data-owner", ownerRow ? "1" : "0");
        var $avatar;
        if (avatar) {
          $avatar = $('<img class="sns-participant-settings-avatar" alt="">').attr("src", avatar);
        } else {
          $avatar = $('<span class="sns-participant-settings-avatar"></span>').text((displayName.charAt(0) || "?").toUpperCase());
        }
        $row.append($avatar);
        $('<span class="sns-participant-settings-name"></span>').text(displayName).appendTo($row);
        $('<input type="color" class="sns-participant-color">').attr("title", "\u0426\u0432\u0435\u0442 \u0438\u043c\u0435\u043d\u0438 \u0432 \u0447\u0430\u0442\u0435").val(participantNameColor(participant, currentConfig || DEFAULTS)).appendTo($row);
        if (ownerRow) {
          $('<span class="sns-participant-owner-badge">\u0441\u043e\u0437\u0434\u0430\u0442\u0435\u043b\u044c</span>').appendTo($row);
        } else if (isOwner()) {
          $('<button type="button" class="sns-participant-remove" title="\u0423\u0434\u0430\u043b\u0438\u0442\u044c">&times;</button>').appendTo($row);
        }
        if (!participant.id || !avatar) {
          resolveParticipantProfile(participant, function(data) {
            if (!data || data.found === false) {
              return;
            }
            var rowStillHere = $.contains(document, $row.get(0));
            if (!rowStillHere) {
              return;
            }
            if (data.id) {
              $row.attr("data-id", data.id);
            }
            if (data.name) {
              $row.attr("data-name", data.name).find(".sns-participant-settings-name").text(data.name);
            }
            if (data.avatar) {
              $row.find(".sns-participant-settings-avatar").first().replaceWith($('<img class="sns-participant-settings-avatar" alt="">').attr("src", data.avatar));
            }
          });
        }
        return $row;
      }
      function renderParticipantsEditor(config) {
        config = config || currentConfig || readConfig();
        var owner = ownerParticipant();
        var participants = effectiveParticipants(config);
        var $editor = $("#sns-participants-editor").empty();
        if (owner.name || owner.id) {
          $editor.append(createParticipantSettingsRow(owner, true));
        }
        participants.forEach(function(participant) {
          participant = normalizeParticipant(participant);
          var sameOwner = owner.id && participant.id && owner.id === participant.id || !owner.id && !participant.id && participantKey(owner.name) === participantKey(participant.name);
          if (sameOwner) {
            return;
          }
          $editor.append(createParticipantSettingsRow(participant, false));
        });
        $("#sns-participant-add-row").toggle(isOwner());
        $("#sns-participants-note").text(isOwner() ? "\u0412\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u043f\u0440\u044f\u043c\u0443\u044e \u0441\u0441\u044b\u043b\u043a\u0443 \u043d\u0430 \u043f\u0440\u043e\u0444\u0438\u043b\u044c \u0443\u0447\u0430\u0441\u0442\u043d\u0438\u043a\u0430. SNS \u0441\u0440\u0430\u0437\u0443 \u0432\u043e\u0437\u044c\u043c\u0435\u0442 RusFF ID \u0438 \u043f\u043e\u0434\u0442\u044f\u043d\u0435\u0442 \u043d\u0438\u043a \u0438 \u0430\u0432\u0430\u0442\u0430\u0440. \u0422\u043e\u043b\u044c\u043a\u043e \u0441\u043e\u0437\u0434\u0430\u0442\u0435\u043b\u044c \u043c\u0435\u043d\u044f\u0435\u0442 \u0441\u043e\u0441\u0442\u0430\u0432." : "\u0421\u043e\u0441\u0442\u0430\u0432 \u0447\u0430\u0442\u0430 \u043c\u043e\u0436\u0435\u0442 \u043c\u0435\u043d\u044f\u0442\u044c \u0442\u043e\u043b\u044c\u043a\u043e \u0435\u0433\u043e \u0441\u043e\u0437\u0434\u0430\u0442\u0435\u043b\u044c.");
      }
      function configFromForm() {
        var ownerControlsParticipants = isOwner();
        var participantEntries = ownerControlsParticipants ? participantEntriesFromEditor() : Array.isArray(currentConfig && currentConfig.participants) ? currentConfig.participants.slice() : [];
        var participantsConfigured = ownerControlsParticipants ? true : !!(currentConfig && currentConfig.participantsConfigured);
        return {
          title: String($("#sns-set-title").val() || "").trim(),
          participantsConfigured: participantsConfigured,
          participantsVersion: 2,
          participants: participantEntries,
          participantColors: participantColorsFromEditor(),
          chatAvatar: String($("#sns-set-avatar").val() || "").trim(),
          avatarGrayscale: clamp($("#sns-set-avatar-gray").val(), 0, 100),
          avatarNoiseOpacity: clamp($("#sns-set-avatar-noise").val(), 0, 100),
          avatarTint: safeColor($("#sns-set-avatar-tint").val(), DEFAULTS.avatarTint),
          avatarTintOpacity: clamp($("#sns-set-avatar-tint-opacity").val(), 0, 80),
          avatarBlendMode: safeChoice($("#sns-set-avatar-blend").val(), [ "normal", "multiply", "soft-light", "color" ], DEFAULTS.avatarBlendMode),
          backgroundUrl: String($("#sns-set-bg").val() || "").trim(),
          overlayColor: safeColor($("#sns-set-overlay").val(), DEFAULTS.overlayColor),
          overlayOpacity: clamp($("#sns-set-opacity").val(), 0, 90),
          noiseOpacity: clamp($("#sns-set-noise").val(), 0, 100),
          ownGradient1: safeColor($("#sns-set-g1").val(), DEFAULTS.ownGradient1),
          ownGradient2: safeColor($("#sns-set-g2").val(), DEFAULTS.ownGradient2),
          gradientAngle: clamp($("#sns-set-angle").val(), 0, 360),
          otherBubble: safeColor($("#sns-set-other").val(), DEFAULTS.otherBubble),
          dialogMode: safeChoice($("#sns-set-dialog-mode").val(), [ "solid", "gradient", "photo", "pattern" ], DEFAULTS.dialogMode),
          dialogColor: safeColor($("#sns-set-dialog-color").val(), DEFAULTS.dialogColor),
          dialogGradient1: safeColor($("#sns-set-dialog-g1").val(), DEFAULTS.dialogGradient1),
          dialogGradient2: safeColor($("#sns-set-dialog-g2").val(), DEFAULTS.dialogGradient2),
          dialogGradientAngle: clamp($("#sns-set-dialog-angle").val(), 0, 360),
          dialogPhotoUrl: String($("#sns-set-dialog-photo").val() || "").trim(),
          dialogPhotoTint: safeColor($("#sns-set-dialog-photo-tint").val(), DEFAULTS.dialogPhotoTint),
          dialogPhotoTintOpacity: clamp($("#sns-set-dialog-photo-opacity").val(), 0, 80),
          dialogNoiseOpacity: clamp($("#sns-set-dialog-noise").val(), 0, 100),
          dialogBlur: clamp($("#sns-set-dialog-blur").val(), 0, 18),
          dialogPattern: safeChoice($("#sns-set-dialog-pattern").val(), [ "dots", "grid", "diagonal", "custom" ], DEFAULTS.dialogPattern),
          dialogPatternUrl: String($("#sns-set-dialog-pattern-url").val() || "").trim(),
          dialogPatternBase: safeColor($("#sns-set-dialog-pattern-base").val(), DEFAULTS.dialogPatternBase),
          dialogPatternColor: safeColor($("#sns-set-dialog-pattern-color").val(), DEFAULTS.dialogPatternColor),
          dialogPatternSize: clamp($("#sns-set-dialog-pattern-size").val(), 10, 160)
        };
      }
      function updateDialogFieldVisibility() {
        var mode = String($("#sns-set-dialog-mode").val() || DEFAULTS.dialogMode);
        $(".sns-dialog-option").addClass("is-hidden").filter('[data-dialog-mode="' + mode + '"]').removeClass("is-hidden");
        var pattern = String($("#sns-set-dialog-pattern").val() || DEFAULTS.dialogPattern);
        $(".sns-pattern-built-in").toggleClass("is-hidden", mode !== "pattern" || pattern === "custom");
        $(".sns-pattern-custom").toggleClass("is-hidden", mode !== "pattern" || pattern !== "custom");
      }
      function updateSettingsPreview() {
        var cfg = configFromForm();
        var $preview = $("#sns-settings-preview");
        $("#sns-overlay-value").text(cfg.overlayOpacity);
        $("#sns-noise-value").text(cfg.noiseOpacity);
        $("#sns-angle-value").text(cfg.gradientAngle);
        $("#sns-dialog-angle-value").text(cfg.dialogGradientAngle);
        $("#sns-dialog-photo-opacity-value").text(cfg.dialogPhotoTintOpacity);
        $("#sns-dialog-noise-value").text(cfg.dialogNoiseOpacity);
        $("#sns-dialog-blur-value").text(cfg.dialogBlur);
        $("#sns-dialog-pattern-size-value").text(cfg.dialogPatternSize);
        $("#sns-avatar-gray-value").text(cfg.avatarGrayscale);
        $("#sns-avatar-noise-value").text(cfg.avatarNoiseOpacity);
        $("#sns-avatar-tint-opacity-value").text(cfg.avatarTintOpacity);
        $("#sns-preview-title").text(cfg.title || originalTopicTitle());
        updateDialogFieldVisibility();
        var previewNode = $preview.get(0);
        if (previewNode && previewNode.style) {
          previewNode.style.setProperty("--preview-overlay", cfg.overlayColor);
          previewNode.style.setProperty("--preview-opacity", String(cfg.overlayOpacity / 100));
          previewNode.style.setProperty("--preview-noise", String(cfg.noiseOpacity / 100));
          previewNode.style.setProperty("--preview-g1", cfg.ownGradient1);
          previewNode.style.setProperty("--preview-g2", cfg.ownGradient2);
          previewNode.style.setProperty("--preview-angle", String(cfg.gradientAngle) + "deg");
          if (cfg.backgroundUrl) {
            previewNode.style.backgroundImage = escapedCssUrl(cfg.backgroundUrl);
          } else {
            previewNode.style.backgroundImage = "none";
          }
        }
        var previewAvatar = cfg.chatAvatar || getPostAvatar(getFirstPost()) || "";
        var avatarNode = $("#sns-preview-avatar").get(0);
        if (avatarNode && avatarNode.style) {
          avatarNode.style.setProperty("--preview-avatar-image", previewAvatar ? escapedCssUrl(previewAvatar) : "none");
          avatarNode.style.setProperty("--preview-avatar-gray", String(cfg.avatarGrayscale) + "%");
          avatarNode.style.setProperty("--preview-avatar-noise", String(cfg.avatarNoiseOpacity / 100));
          avatarNode.style.setProperty("--preview-avatar-tint", cfg.avatarTint);
          avatarNode.style.setProperty("--preview-avatar-tint-opacity", String(cfg.avatarTintOpacity / 100));
          avatarNode.style.setProperty("--preview-avatar-blend", cfg.avatarBlendMode);
        }
        var dialogBg = dialogBackgroundCss(cfg);
        var dialogNode = $preview.find(".sns-preview-dialog").get(0);
        if (dialogNode && dialogNode.style) {
          dialogNode.style.setProperty("--preview-dialog-color", dialogBg.color);
          dialogNode.style.setProperty("--preview-dialog-image", dialogBg.image);
          dialogNode.style.setProperty("--preview-dialog-size", dialogBg.size);
          dialogNode.style.setProperty("--preview-dialog-repeat", dialogBg.repeat);
          dialogNode.style.setProperty("--preview-dialog-position", dialogBg.position);
          dialogNode.style.setProperty("--preview-dialog-noise", String(cfg.dialogNoiseOpacity / 100));
          dialogNode.style.setProperty("--preview-dialog-blur", String(cfg.dialogBlur) + "px");
        }
      }
      function fillSettings(config) {
        $("#sns-set-title").val(config.title || "");
        $("#sns-set-avatar").val(config.chatAvatar || "");
        $("#sns-set-avatar-gray").val(clamp(config.avatarGrayscale, 0, 100));
        $("#sns-set-avatar-noise").val(clamp(config.avatarNoiseOpacity, 0, 100));
        $("#sns-set-avatar-tint").val(safeColor(config.avatarTint, DEFAULTS.avatarTint));
        $("#sns-set-avatar-tint-opacity").val(clamp(config.avatarTintOpacity, 0, 80));
        $("#sns-set-avatar-blend").val(safeChoice(config.avatarBlendMode, [ "normal", "multiply", "soft-light", "color" ], DEFAULTS.avatarBlendMode));
        $("#sns-set-bg").val(config.backgroundUrl || "");
        $("#sns-set-overlay").val(safeColor(config.overlayColor, DEFAULTS.overlayColor));
        $("#sns-set-opacity").val(clamp(config.overlayOpacity, 0, 90));
        $("#sns-set-noise").val(clamp(config.noiseOpacity, 0, 100));
        $("#sns-set-g1").val(safeColor(config.ownGradient1, DEFAULTS.ownGradient1));
        $("#sns-set-g2").val(safeColor(config.ownGradient2, DEFAULTS.ownGradient2));
        $("#sns-set-angle").val(clamp(config.gradientAngle, 0, 360));
        $("#sns-set-other").val(safeColor(config.otherBubble, DEFAULTS.otherBubble));
        $("#sns-set-dialog-mode").val(safeChoice(config.dialogMode, [ "solid", "gradient", "photo", "pattern" ], DEFAULTS.dialogMode));
        $("#sns-set-dialog-color").val(safeColor(config.dialogColor, DEFAULTS.dialogColor));
        $("#sns-set-dialog-g1").val(safeColor(config.dialogGradient1, DEFAULTS.dialogGradient1));
        $("#sns-set-dialog-g2").val(safeColor(config.dialogGradient2, DEFAULTS.dialogGradient2));
        $("#sns-set-dialog-angle").val(clamp(config.dialogGradientAngle, 0, 360));
        $("#sns-set-dialog-photo").val(config.dialogPhotoUrl || "");
        $("#sns-set-dialog-photo-tint").val(safeColor(config.dialogPhotoTint, DEFAULTS.dialogPhotoTint));
        $("#sns-set-dialog-photo-opacity").val(clamp(config.dialogPhotoTintOpacity, 0, 80));
        $("#sns-set-dialog-noise").val(clamp(config.dialogNoiseOpacity, 0, 100));
        $("#sns-set-dialog-blur").val(clamp(config.dialogBlur, 0, 18));
        $("#sns-set-dialog-pattern").val(safeChoice(config.dialogPattern, [ "dots", "grid", "diagonal", "custom" ], DEFAULTS.dialogPattern));
        $("#sns-set-dialog-pattern-url").val(config.dialogPatternUrl || "");
        $("#sns-set-dialog-pattern-base").val(safeColor(config.dialogPatternBase, DEFAULTS.dialogPatternBase));
        $("#sns-set-dialog-pattern-color").val(safeColor(config.dialogPatternColor, DEFAULTS.dialogPatternColor));
        $("#sns-set-dialog-pattern-size").val(clamp(config.dialogPatternSize, 10, 160));
        $("#sns-settings-status").text("");
        renderParticipantsEditor(config);
        updateDialogFieldVisibility();
        updateSettingsPreview();
      }
      function openSettings() {
        if (!canEditAppearance(currentConfig)) {
          return;
        }
        buildSettingsModal();
        fillSettings(currentConfig || readConfig());
        $("#sns-settings-backdrop").addClass("is-open");
      }
      function closeSettings() {
        $("#sns-settings-backdrop").removeClass("is-open");
        applyParticipantNameColors(currentConfig);
      }
      function findEditLink($post) {
        var $result = $();
        $post.find(".post-links a").each(function() {
          var $link = $(this);
          var text = cleanText($link.text());
          var href = $link.attr("href") || "";
          if (/\u0440\u0435\u0434\u0430\u043A\u0442|edit/i.test(text) || /edit\.php|action=edit/i.test(href)) {
            $result = $link;
            return false;
          }
        });
        return $result;
      }
      function persistConfig(config, callback) {
        var $nativeForm = $("#post").first();
        var $nativeReply = $("#main-reply").first();
        var $ui = $("#sns-ui-input").first();
        if (!$nativeForm.length || !$nativeReply.length) {
          callback(false, "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043d\u0430\u0439\u0442\u0438 \u0444\u043e\u0440\u043c\u0443 \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0438.");
          return;
        }
        var encoded = encodeConfig(config);
        var payload = CONFIG_PREFIX + encoded;
        var nativeBefore = String($nativeReply.val() || "");
        var uiBefore = $ui.length ? String($ui.val() || "") : "";
        var finished = false;
        var timeout = null;
        var verifyTimers = [];
        var verifyRequest = null;
        var $submit = $nativeForm.find('input[type="submit"][name="submit"],button[type="submit"][name="submit"]').first();
        if (!$submit.length) {
          $submit = $nativeForm.find('input[type="submit"],button[type="submit"]').first();
        }
        if (!$submit.length) {
          callback(false, "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043d\u0430\u0439\u0442\u0438 \u043a\u043d\u043e\u043f\u043a\u0443 \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0438.");
          return;
        }
        function restoreComposer() {
          setTimeout(function() {
            $nativeReply.val(nativeBefore).trigger("input").trigger("change");
            if ($ui.length) {
              $ui.val(uiBefore).trigger("input");
            }
          }, 90);
        }
        function clearVerification() {
          verifyTimers.forEach(function(timer) {
            clearTimeout(timer);
          });
          verifyTimers = [];
          if (verifyRequest && typeof verifyRequest.abort === "function") {
            try {
              verifyRequest.abort();
            } catch (error) {}
          }
          verifyRequest = null;
        }
        function finish(ok, message) {
          if (finished) {
            return;
          }
          finished = true;
          if (timeout) {
            clearTimeout(timeout);
          }
          clearVerification();
          $(document).off("pun_post.snsConfigPersist").off("sns_forum_publish_confirmed.snsConfigPersist");
          if (ok) {
            cacheConfigEncoded(encoded, {
              source: "confirmed-save",
              savedAt: configSavedAtValue(config),
              cachedAt: Date.now()
            });
          }
          restoreComposer();
          setTimeout(function() {
            if (typeof enhancePosts === "function") {
              enhancePosts();
            }
            getTopic().find(".post").each(function() {
              var raw = cleanText($(this).find(".post-content").first().text());
              if (/SNSCFG:[A-Za-z0-9+\/=]+/.test(raw)) {
                $(this).addClass("sns-config-post").attr("aria-hidden", "true");
              }
            });
          }, 120);
          callback(ok, message);
        }
        function messageContainsPayload(message) {
          var holder = document.createElement("div");
          holder.innerHTML = String(message || "");
          var raw = String(holder.textContent || holder.innerText || "");
          return raw.indexOf(payload) !== -1;
        }
        function payloadExistsInDom() {
          var found = false;
          getTopic().find(".post").each(function() {
            var raw = cleanText($(this).find(".post-content").first().text());
            if (raw.indexOf(payload) !== -1) {
              found = true;
              return false;
            }
          });
          return found;
        }
        function verifyWithApi() {
          if (finished) {
            return;
          }
          if (payloadExistsInDom()) {
            finish(true, "\u041d\u0430\u0441\u0442\u0440\u043e\u0439\u043a\u0438 \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u044b.");
            return;
          }
          var topicId = snsTopicId();
          if (!topicId) {
            return;
          }
          var apiUrl = new URL("api.php", location.href);
          apiUrl.searchParams.set("method", "post.get");
          apiUrl.searchParams.set("topic_id", topicId);
          apiUrl.searchParams.set("sort_by", "id");
          apiUrl.searchParams.set("sort_dir", "desc");
          apiUrl.searchParams.set("limit", "20");
          apiUrl.searchParams.set("fields", "id,message,topic_id,user_id");
          apiUrl.searchParams.set("_sns_config_verify", String(Date.now()));
          verifyRequest = SNSRequest({
            url: apiUrl.toString(),
            type: "GET",
            dataType: "json",
            cache: false,
            timeout: 4e3
          }).done(function(json) {
            if (finished) {
              return;
            }
            var posts = json && Array.isArray(json.response) ? json.response : [];
            var found = posts.some(function(post) {
              return post && messageContainsPayload(post.message);
            });
            if (found) {
              finish(true, "\u041d\u0430\u0441\u0442\u0440\u043e\u0439\u043a\u0438 \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u044b.");
            }
          }).always(function() {
            verifyRequest = null;
          });
        }
        $(document).one("pun_post.snsConfigPersist", function() {
          finish(true, "\u041d\u0430\u0441\u0442\u0440\u043e\u0439\u043a\u0438 \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u044b.");
        });
        $(document).one("sns_forum_publish_confirmed.snsConfigPersist", function() {
          finish(true, "\u041d\u0430\u0441\u0442\u0440\u043e\u0439\u043a\u0438 \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u044b.");
        });
        [ 450, 1100, 2200, 4e3, 6500 ].forEach(function(delay) {
          verifyTimers.push(setTimeout(verifyWithApi, delay));
        });
        timeout = setTimeout(function() {
          finish(false, "\u041d\u0430\u0441\u0442\u0440\u043e\u0439\u043a\u0438 \u043f\u0440\u0438\u043c\u0435\u043d\u0435\u043d\u044b \u043d\u0430 \u044d\u0442\u043e\u0439 \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u0435, \u043d\u043e \u0441\u0435\u0440\u0432\u0435\u0440\u043d\u0430\u044f \u0437\u0430\u043f\u0438\u0441\u044c \u043f\u043e\u043a\u0430 \u043d\u0435 \u043f\u043e\u0434\u0442\u0432\u0435\u0440\u0436\u0434\u0435\u043d\u0430.");
        }, 9e3);
        $nativeReply.val(payload).trigger("input").trigger("change");
        $submit.trigger("click");
      }
      function extractImageUrl(text) {
        text = String(text || "");
        var urlTag = text.match(/\[url=(https?:\/\/[^\]\s]+)\]/i);
        if (urlTag) return urlTag[1];
        var imgTag = text.match(/\[img\](https?:\/\/[^\[]+)\[\/img\]/i);
        if (imgTag) return imgTag[1].trim();
        var bare = text.match(/https?:\/\/[^\s\]]+\.(?:png|jpe?g|gif|webp)(?:\?[^\s\]]*)?/i);
        return bare ? bare[0] : "";
      }
      function ensureSettingsPopoverClose($area) {
        if (!$area || !$area.length) {
          return;
        }
        if (!$area.children(".sns-settings-popover-close").length) {
          $area.prepend('<button type="button" class="sns-settings-popover-close" aria-label="Close">&times;</button>');
        }
      }
      function findSettingsImageArea() {
        var $areas = $('[id="image-area"]');
        if (!$areas.length) {
          return $();
        }
        var $fresh = $areas.filter(function() {
          return !$(this).hasClass("sns-settings-upload-popover");
        }).last();
        return $fresh.length ? $fresh : $areas.last();
      }
      function positionSettingsUploadPopover(triggerElement) {
        var $area = findSettingsImageArea();
        if (!$area.length) {
          return false;
        }
        var node = $area.get(0);
        if (!uploadCapture || uploadCapture.areaNode !== node) {
          if (uploadCapture) {
            uploadCapture.areaNode = node;
          }
          $area.detach().appendTo(document.body);
        }
        var rect = triggerElement && triggerElement.getBoundingClientRect ? triggerElement.getBoundingClientRect() : null;
        var width = Math.min(360, Math.max(280, window.innerWidth - 24));
        $area.removeClass("sns-settings-upload-area sns-image-area").addClass("sns-settings-upload-popover").show();
        ensureSettingsPopoverClose($area);
        var estimatedHeight = Math.min(260, Math.max(150, $area.outerHeight() || 210));
        var left = rect ? Math.round(rect.right - width) : Math.round((window.innerWidth - width) / 2);
        left = Math.max(12, Math.min(left, window.innerWidth - width - 12));
        var top = rect ? Math.round(rect.bottom + 8) : Math.round((window.innerHeight - estimatedHeight) / 2);
        if (top + estimatedHeight > window.innerHeight - 12) {
          top = rect ? Math.round(rect.top - estimatedHeight - 8) : Math.round((window.innerHeight - estimatedHeight) / 2);
        }
        top = Math.max(12, Math.min(top, window.innerHeight - estimatedHeight - 12));
        document.body.style.setProperty("--sns-upload-left", left + "px");
        document.body.style.setProperty("--sns-upload-top", top + "px");
        $("body").removeClass("sns-settings-upload-pending");
        return true;
      }
      function closeSettingsUploadPopover() {
        $("body").removeClass("sns-settings-upload-pending");
        var $area = $('body > [id="image-area"].sns-settings-upload-popover').last();
        if (!$area.length) {
          $area = findSettingsImageArea();
        }
        if ($area.length) {
          $area.removeClass("sns-settings-upload-popover").addClass("sns-image-area").hide();
          var $shell = $("#sns-chat-shell");
          if ($shell.length) {
            $area.detach().appendTo($shell);
          }
        }
      }
      function stopUploadCapture(restore) {
        if (!uploadCapture) {
          closeSettingsUploadPopover();
          return;
        }
        clearInterval(uploadCapture.timer);
        if (restore) {
          $("#main-reply").val(uploadCapture.nativeBefore).trigger("input").trigger("change");
          $("#sns-ui-input").val(uploadCapture.uiBefore).trigger("input");
        }
        closeSettingsUploadPopover();
        uploadCapture = null;
      }
      function beginSettingsUpload(targetId, triggerElement) {
        if (uploadCapture) {
          stopUploadCapture(true);
        }
        var $native = $("#main-reply").first();
        var $ui = $("#sns-ui-input").first();
        var $photoAction = $(".sns-attach-photo").first();
        if (!$native.length || !$ui.length || !$photoAction.length) {
          $("#sns-settings-status").text("\u0417\u0430\u0433\u0440\u0443\u0437\u0447\u0438\u043a \u0444\u043e\u0440\u0443\u043c\u0430 \u043d\u0435 \u043d\u0430\u0439\u0434\u0435\u043d.");
          return;
        }
        uploadCapture = {
          targetId: targetId,
          nativeBefore: $native.val(),
          uiBefore: $ui.val(),
          triggerElement: triggerElement || null,
          tries: 0,
          timer: null
        };
        $native.val("").trigger("input").trigger("change");
        $ui.val("").trigger("input");
        $("body").addClass("sns-settings-upload-pending");
        $photoAction.trigger("click");
        function tryPosition(attempt) {
          if (!uploadCapture) {
            return;
          }
          if (positionSettingsUploadPopover(uploadCapture.triggerElement)) {
            return;
          }
          if (attempt < 30) {
            setTimeout(function() {
              tryPosition(attempt + 1);
            }, 25);
          } else {
            $("body").removeClass("sns-settings-upload-pending");
          }
        }
        setTimeout(function() {
          tryPosition(0);
        }, 0);
        uploadCapture.timer = setInterval(function() {
          if (!uploadCapture) {
            return;
          }
          uploadCapture.tries++;
          var $area = findSettingsImageArea();
          if ($area.length && (!$area.hasClass("sns-settings-upload-popover") || uploadCapture.areaNode !== $area.get(0))) {
            positionSettingsUploadPopover(uploadCapture.triggerElement);
          }
          var raw = String($native.val() || $ui.val() || "");
          var url = extractImageUrl(raw);
          if (url) {
            $("#" + uploadCapture.targetId).val(url).trigger("input");
            updateSettingsPreview();
            stopUploadCapture(true);
            $("#sns-settings-status").text("\u0418\u0437\u043e\u0431\u0440\u0430\u0436\u0435\u043d\u0438\u0435 \u0434\u043e\u0431\u0430\u0432\u043b\u0435\u043d\u043e.");
            return;
          }
          if (uploadCapture.tries > 1200) {
            stopUploadCapture(true);
            $("#sns-settings-status").text("\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u043e\u0442\u043c\u0435\u043d\u0435\u043d\u0430.");
          }
        }, 100);
      }
      function profileUrlForPost($post) {
        var userId = getPostUserId($post);
        if (userId) {
          return new URL("profile.php?id=" + encodeURIComponent(userId), location.href).toString();
        }
        var href = $post.find('.pa-author a[href*="profile.php?id="]').first().attr("href") || "";
        if (!href) {
          return "";
        }
        return new URL(href, location.href).toString();
      }
      function openSnsProfileFromElement(element) {
        var $element = $(element);
        var participantId = String($element.attr("data-participant-id") || "");
        var url = "";
        if (/^\d+$/.test(participantId)) {
          url = new URL("profile.php?id=" + encodeURIComponent(participantId), location.href).toString();
        } else {
          var $post = $element.closest(".post");
          if ($post.length) {
            url = profileUrlForPost($post);
          }
        }
        if (!url) {
          return false;
        }
        location.href = url;
        return true;
      }
      function bindEvents() {
        $(document).off(".snsCustomV07").on("click.snsCustomV07", ".sns-settings-open", function(event) {
          event.preventDefault();
          event.stopPropagation();
          var $button = $(this);
          $button.removeClass("is-spinning");
          void this.offsetWidth;
          $button.addClass("is-spinning");
          clearTimeout($button.data("snsGearOpenTimer"));
          var timer = setTimeout(function() {
            $button.removeClass("is-spinning");
            openSettings();
          }, 500);
          $button.data("snsGearOpenTimer", timer);
        }).on("click.snsCustomV07", ".sns-author-name[data-sns-profile-link], .sns-mini-avatar[data-sns-profile-link], .sns-participant-avatar[data-participant-id]", function(event) {
          event.preventDefault();
          event.stopPropagation();
          openSnsProfileFromElement(this);
        }).on("keydown.snsCustomV07", ".sns-author-name[data-sns-profile-link], .sns-mini-avatar[data-sns-profile-link], .sns-participant-avatar[data-participant-id]", function(event) {
          if (event.key !== "Enter" && event.key !== " ") {
            return;
          }
          event.preventDefault();
          event.stopPropagation();
          openSnsProfileFromElement(this);
        }).on("click.snsCustomV07", ".sns-settings-tab", function() {
          var key = String($(this).attr("data-settings-tab") || "");
          if (!key) return;
          $(".sns-settings-tab").removeClass("is-active");
          $(this).addClass("is-active");
          $(".sns-settings-pane").removeClass("is-active").filter('[data-settings-pane="' + key + '"]').addClass("is-active");
        }).on("click.snsCustomV07", ".sns-settings-close, #sns-settings-cancel", function() {
          closeSettings();
        }).on("click.snsCustomV07", "#sns-settings-backdrop", function(event) {
          if (event.target === this) {
            closeSettings();
          }
        }).on("input.snsCustomV07 change.snsCustomV07", "#sns-settings-modal input, #sns-settings-modal select", function() {
          updateSettingsPreview();
          if ($(this).hasClass("sns-participant-color")) {
            var liveConfig = $.extend({}, currentConfig || DEFAULTS, {
              participantColors: participantColorsFromEditor()
            });
            applyParticipantNameColors(liveConfig);
          }
        }).on("click.snsCustomV07", ".sns-participant-add", function() {
          if (!isOwner()) {
            return;
          }
          var $input = $("#sns-participant-name");
          var profileValue = String($input.val() || "").trim();
          if (!profileValue) {
            return;
          }
          var idMatch = profileValue.match(/profile\.php\?[^#]*\bid=(\d+)/i);
          if (!idMatch) {
            $("#sns-settings-status").text("\u041d\u0443\u0436\u043d\u0430 \u0441\u0441\u044b\u043b\u043a\u0430 \u0432\u0438\u0434\u0430 profile.php?id=123");
            return;
          }
          var participantId = String(idMatch[1]);
          try {
            if (/^https?:\/\//i.test(profileValue)) {
              var parsedProfileUrl = new URL(profileValue);
              if (parsedProfileUrl.host !== location.host) {
                $("#sns-settings-status").text("\u0421\u0441\u044b\u043b\u043a\u0430 \u0434\u043e\u043b\u0436\u043d\u0430 \u0432\u0435\u0441\u0442\u0438 \u043d\u0430 \u043f\u0440\u043e\u0444\u0438\u043b\u044c \u044d\u0442\u043e\u0433\u043e \u0444\u043e\u0440\u0443\u043c\u0430.");
                return;
              }
            }
          } catch (error) {
            $("#sns-settings-status").text("\u041d\u0435\u043a\u043e\u0440\u0440\u0435\u043a\u0442\u043d\u0430\u044f \u0441\u0441\u044b\u043b\u043a\u0430 \u043d\u0430 \u043f\u0440\u043e\u0444\u0438\u043b\u044c.");
            return;
          }
          var owner = ownerParticipant();
          if (owner.id && String(owner.id) === participantId) {
            $input.val("");
            $("#sns-settings-status").text("\u0421\u043e\u0437\u0434\u0430\u0442\u0435\u043b\u044c \u0443\u0436\u0435 \u0435\u0441\u0442\u044c \u0432 \u0447\u0430\u0442\u0435.");
            return;
          }
          var exists = false;
          $("#sns-participants-editor").children(".sns-participant-settings-row").each(function() {
            var rowId = String($(this).attr("data-id") || "");
            if (rowId && rowId === participantId) {
              exists = true;
              return false;
            }
          });
          if (exists) {
            $("#sns-settings-status").text("\u042d\u0442\u043e\u0442 \u0443\u0447\u0430\u0441\u0442\u043d\u0438\u043a \u0443\u0436\u0435 \u0435\u0441\u0442\u044c \u0432 \u0447\u0430\u0442\u0435.");
            return;
          }
          var item = {
            id: participantId,
            name: ""
          };
          var $row = createParticipantSettingsRow(item, false);
          $("#sns-participants-editor").append($row);
          $input.val("");
          $("#sns-settings-status").text("ID " + participantId + " \u0434\u043e\u0431\u0430\u0432\u043b\u0435\u043d. \u041d\u0438\u043a \u0438 \u0430\u0432\u0430\u0442\u0430\u0440 \u043f\u043e\u0434\u0442\u044f\u0433\u0438\u0432\u0430\u044e\u0442\u0441\u044f\u2026");
          resolveParticipantProfile(item, function(data) {
            if (!data || !$.contains(document, $row.get(0))) {
              return;
            }
            var displayName = cleanText(data.name || "");
            if (data.id) {
              $row.attr("data-id", String(data.id));
            }
            if (displayName) {
              $row.attr("data-name", displayName).find(".sns-participant-settings-name").text(displayName);
            }
            if (data.avatar) {
              $row.find(".sns-participant-settings-avatar").first().replaceWith($('<img class="sns-participant-settings-avatar" alt="">').attr("src", data.avatar));
            }
            if (displayName || data.avatar) {
              $("#sns-settings-status").text((displayName ? displayName + " \xb7 " : "") + "ID " + participantId + " \u0433\u043e\u0442\u043e\u0432");
            } else {
              $("#sns-settings-status").text("ID " + participantId + " \u0434\u043e\u0431\u0430\u0432\u043b\u0435\u043d. \u041f\u0440\u043e\u0444\u0438\u043b\u044c \u043d\u0435 \u043e\u0442\u0432\u0435\u0442\u0438\u043b \u0432\u043e\u0432\u0440\u0435\u043c\u044f; \u0434\u043e\u0441\u0442\u0443\u043f \u0432\u0441\u0435 \u0440\u0430\u0432\u043d\u043e \u0440\u0430\u0431\u043e\u0442\u0430\u0435\u0442.");
            }
          });
        }).on("keydown.snsCustomV07", "#sns-participant-name", function(event) {
          if (event.key === "Enter") {
            event.preventDefault();
            $(".sns-participant-add").trigger("click");
          }
        }).on("click.snsCustomV07", ".sns-participant-remove", function() {
          if (!isOwner()) {
            return;
          }
          $(this).closest(".sns-participant-settings-row").remove();
          $("#sns-settings-status").text("");
        }).on("click.snsCustomV07", "#sns-settings-save", function() {
          if (!canEditAppearance(currentConfig)) {
            closeSettings();
            return;
          }
          var cfg = configFromForm();
          cfg.configSchema = 67;
          cfg.configSavedAt = Date.now();
          var $button = $(this);
          $button.prop("disabled", true);
          $("#sns-settings-status").text("\u0421\u043e\u0445\u0440\u0430\u043d\u044f\u044e...");
          applyConfig(cfg);
          persistConfig(cfg, function(ok, message) {
            $button.prop("disabled", false);
            $("#sns-settings-status").text(message);
            if (ok) {
              var encoded = encodeConfig(cfg);
              cacheConfigEncoded(encoded, {
                source: "confirmed-save",
                savedAt: configSavedAtValue(cfg),
                cachedAt: Date.now()
              });
              installRemoteConfigShadow(encoded);
              currentConfig = cfg;
              setTimeout(closeSettings, 450);
            }
          });
        }).on("pun_post.snsCustomV07 pun_edit.snsCustomV07", function() {
          setTimeout(function() {
            currentConfig = readConfig();
            applyConfig(currentConfig);
          }, 140);
        });
      }
      var dialogBackgroundResizeTimer = null;
      $(window).off("resize.snsDialogBackground").on("resize.snsDialogBackground", function() {
        clearTimeout(dialogBackgroundResizeTimer);
        dialogBackgroundResizeTimer = setTimeout(positionDialogBackgroundLayer, 80);
      });
      function resetSettingsUploaderArtifacts() {
        $("body").removeClass("sns-settings-upload-pending");
        var $area = $('[id="image-area"]').last();
        if (!$area.length) {
          return;
        }
        $area.removeClass("sns-settings-upload-popover sns-settings-upload-area").addClass("sns-image-area").hide();
        var $shell = $("#sns-chat-shell");
        if ($shell.length && !$area.parent().is($shell)) {
          $area.detach().appendTo($shell);
        }
      }
      $(document).off("sns_pages_loaded.snsParticipantsV44").on("sns_pages_loaded.snsParticipantsV44", function() {
        setTimeout(function() {
          renderHeaderTools();
          applyAccessControl();
          var $topic = getTopic();
          if ($topic.length && window.__SNS_LAZY_HISTORY_PREPENDING__ !== true) {
            $topic.stop(true).animate({
              scrollTop: $topic.get(0).scrollHeight
            }, 0);
          }
        }, 120);
      });
      function initCustomization() {
        if (!$("#sns-chat-shell").length || !$("#sns-chat-header").length) return false;
        resetSettingsUploaderArtifacts();
        addCustomStyle();
        buildSettingsModal();
        currentConfig = readConfig();
        applyConfig(currentConfig);
        bindEvents();
        if (!window.__SNS_NAME_COLOR_EVENTS_V238__) {
          window.__SNS_NAME_COLOR_EVENTS_V238__ = true;
          var colorRefreshTimer = null;
          $(document).off("sns_pages_loaded.snsNameColorsV238 " + "pun_edit.snsNameColorsV238").on("sns_pages_loaded.snsNameColorsV238 " + "pun_edit.snsNameColorsV238", function() {
            if (colorRefreshTimer) {
              clearTimeout(colorRefreshTimer);
            }
            colorRefreshTimer = setTimeout(function() {
              colorRefreshTimer = null;
              applyParticipantNameColors(currentConfig);
            }, 120);
          });
        }
        setTimeout(function() {
          positionDialogBackgroundLayer();
          loadLatestConfigFromServer(function(latestConfig) {
            if (!latestConfig) {
              return;
            }
            currentConfig = latestConfig;
            applyConfig(currentConfig);
          });
        }, 80);
        return true;
      }
      if (!initCustomization()) {
        var attempts = 0;
        var timer = setInterval(function() {
          attempts++;
          if (initCustomization() || attempts > 80) {
            clearInterval(timer);
          }
        }, 100);
      }
    })(jQuery);
    (function($) {
      "use strict";
      $(document).off("keydown.snsSendShortcut", "#sns-ui-input").on("keydown.snsSendShortcut", "#sns-ui-input", function(event) {
        if (event.key === "Enter" && (event.ctrlKey || event.metaKey)) {
          event.preventDefault();
          $(".sns-ui-send").first().trigger("click");
        }
      });
    })(jQuery);
    (function($) {
      "use strict";
      if (!$("body").hasClass("sns-chat-page")) {
        return;
      }
      var editState = {
        active: false,
        saving: false,
        $post: $(),
        oldSendHtml: "",
        oldSendTitle: "",
        pollTimer: null,
        apiMode: false,
        apiPostId: "",
        apiEditUrl: "",
        apiForm: null,
        apiTextarea: null,
        apiSubmitName: "",
        apiSubmitValue: "",
        voiceMode: false,
        audioMode: false,
        audioData: null,
        replyMeta: null,
        replyEncoded: "",
        storyEncoded: "",
        apiFrame: null,
        apiFrameTimer: null,
        apiFramePhase: "",
        apiSubmitButton: null,
        apiFormReady: false,
        apiSourceReady: false,
        apiSourceValue: "",
        inlineMounted: false,
        inlineOriginalHtml: "",
        inlineSaveTimer: null,
        nativeReady: false,
        nativeProxyRestoreTimer: null,
        nativeRawLoaded: "",
        nativeReplyBaseline: null,
        nativeReleaseTimers: []
      };
      function cleanText(value) {
        return String(value || "").replace(/\s+/g, " ").trim();
      }
      function uiInput() {
        return $("#sns-ui-input").first();
      }
      function nativeReply() {
        return $("#main-reply").first();
      }
      function nativeQuickEditSubmitButton() {
        var $form = $("#post").first();
        var $button = $form.find('input[type="submit"][name="submit"],' + 'button[type="submit"][name="submit"]').first();
        if (!$button.length) {
          $button = $form.find('input[type="submit"],button[type="submit"]').filter(function() {
            var label = cleanText($(this).val() || $(this).text() || "");
            return !/\u043f\u0440\u0435\u0434\u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440|preview/i.test(label);
          }).first();
        }
        return $button;
      }
      function captureNativeReplyBaseline() {
        var $form = $("#post").first();
        if (!$form.length) {
          return;
        }
        var action = String($form.attr("action") || "");
        if (/edit\.php/i.test(action)) {
          return;
        }
        var $submit = nativeQuickEditSubmitButton();
        editState.nativeReplyBaseline = {
          action: action,
          method: String($form.attr("method") || ""),
          target: String($form.attr("target") || ""),
          submitName: $submit.length ? String($submit.attr("name") || "") : "",
          submitValue: $submit.length ? String($submit.val() || $submit.text() || "") : ""
        };
      }
      function clearNativeReleaseTimers() {
        (editState.nativeReleaseTimers || []).forEach(function(timer) {
          clearTimeout(timer);
        });
        editState.nativeReleaseTimers = [];
      }
      function restoreNativeReplyFormBaseline() {
        var baseline = editState.nativeReplyBaseline;
        var $form = $("#post").first();
        if (!$form.length) {
          return;
        }
        if (baseline) {
          if (baseline.action) {
            $form.attr("action", baseline.action);
          } else {
            $form.removeAttr("action");
          }
          if (baseline.method) {
            $form.attr("method", baseline.method);
          }
          if (baseline.target) {
            $form.attr("target", baseline.target);
          } else {
            $form.removeAttr("target");
          }
          var $submit = nativeQuickEditSubmitButton();
          if ($submit.length) {
            if (baseline.submitName) {
              $submit.attr("name", baseline.submitName);
            }
            if (baseline.submitValue && $submit.is("input")) {
              $submit.val(baseline.submitValue);
            }
          }
        }
        $form.find('input[type="hidden"]').filter(function() {
          var name = String($(this).attr("name") || "");
          return /^(?:edit|edit_id|post_id_to_edit|edit_post_id)$/i.test(name);
        }).remove();
        nativeReply().val("").trigger("input").trigger("change");
      }
      function releaseNativeQuickEditMode() {
        clearNativeReleaseTimers();
        var $cancel = findNativeCancel();
        if ($cancel.length) {
          try {
            $cancel.get(0).click();
          } catch (error) {}
        }
        restoreNativeReplyFormBaseline();
        [ 0, 80, 220 ].forEach(function(delay) {
          editState.nativeReleaseTimers.push(setTimeout(restoreNativeReplyFormBaseline, delay));
        });
      }
      function apiPostEditHref($post) {
        return String($post && $post.length ? $post.find('.post-links a[href*="edit.php"]').first().attr("href") || "" : "");
      }
      function findBoundNativeEditProxy($excludePost) {
        var $result = $();
        $("#pun-viewtopic .post").not('[data-sns-api-post="1"]').each(function() {
          var $post = $(this);
          if ($excludePost && $excludePost.length && $post.get(0) === $excludePost.get(0)) {
            return;
          }
          var $link = $post.find('.post-links a[href*="edit.php"]').first();
          if ($link.length) {
            $result = $link;
            return false;
          }
        });
        return $result;
      }
      function restoreNativeProxyState($proxy, oldHref, $proxyPost, oldProxyPostId, $targetPost, oldTargetPostId) {
        if (editState.nativeProxyRestoreTimer) {
          clearTimeout(editState.nativeProxyRestoreTimer);
        }
        editState.nativeProxyRestoreTimer = setTimeout(function() {
          if ($proxy && $proxy.length) {
            $proxy.attr("href", oldHref);
          }
          if ($proxyPost && $proxyPost.length && oldProxyPostId) {
            $proxyPost.attr("id", oldProxyPostId);
          }
          if ($targetPost && $targetPost.length && oldTargetPostId) {
            $targetPost.attr("id", oldTargetPostId);
          }
          editState.nativeProxyRestoreTimer = null;
        }, 120);
      }
      function launchNativeQuickEditForPost($post) {
        if (!$post || !$post.length) {
          return false;
        }
        editState.apiMode = false;
        editState.nativeReady = false;
        editState.voiceMode = false;
        editState.audioMode = false;
        editState.audioData = null;
        editState.replyEncoded = "";
        editState.storyEncoded = String($post.attr("data-sns-story") || "");
        beginVisualEdit($post, false);
        if ($post.attr("data-sns-api-post") !== "1") {
          return true;
        }
        var targetHref = apiPostEditHref($post);
        var $proxy = findBoundNativeEditProxy($post);
        if (!targetHref || !$proxy.length) {
          resetVisualEdit(false);
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u041d\u0435 \u043d\u0430\u0448\u043b\u0430\u0441\u044c \u0440\u043e\u0434\u043d\u0430\u044f AJAX-\u0441\u0441\u044b\u043b\u043a\u0430 RusFF \u0434\u043b\u044f quick-edit.");
          return false;
        }
        var oldHref = String($proxy.attr("href") || "");
        var $proxyPost = $proxy.closest(".post");
        var oldProxyPostId = String($proxyPost.attr("id") || "");
        var oldTargetPostId = String($post.attr("id") || "");
        var targetNumericId = oldTargetPostId.replace(/^p/i, "").replace(/\D+/g, "");
        if (!targetNumericId) {
          resetVisualEdit(false);
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043e\u043f\u0440\u0435\u0434\u0435\u043b\u0438\u0442\u044c ID \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f.");
          return false;
        }
        $post.attr("id", "sns-edit-target-" + targetNumericId);
        $proxyPost.attr("id", "p" + targetNumericId);
        $proxy.attr("href", targetHref);
        try {
          $proxy.get(0).click();
        } catch (error) {
          $proxy.attr("href", oldHref);
          if (oldProxyPostId) {
            $proxyPost.attr("id", oldProxyPostId);
          }
          if (oldTargetPostId) {
            $post.attr("id", oldTargetPostId);
          }
          resetVisualEdit(false);
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u0437\u0430\u043f\u0443\u0441\u0442\u0438\u0442\u044c \u0440\u043e\u0434\u043d\u043e\u0439 quick-edit RusFF.");
          return false;
        }
        restoreNativeProxyState($proxy, oldHref, $proxyPost, oldProxyPostId, $post, oldTargetPostId);
        return true;
      }
      function sendButton() {
        return $(".sns-ui-send").first();
      }
      function plusButton() {
        return $(".sns-ui-plus").first();
      }
      function ensureEditBanner() {
        var $banner = $("#sns-edit-banner");
        if ($banner.length) {
          return $banner;
        }
        $banner = $('<div id="sns-edit-banner">' + '<div class="sns-edit-banner-main">' + '<span class="sns-edit-pencil">\u270e</span>' + '<div class="sns-edit-banner-copy">' + '<b id="sns-edit-banner-title">\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f</b>' + '<span id="sns-edit-banner-sub">\u0417\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0442\u0435\u043a\u0441\u0442...</span>' + "</div>" + "</div>" + '<button type="button" id="sns-edit-cancel">\u041e\u0442\u043c\u0435\u043d\u0430</button>' + "</div>");
        $("#sns-composer-ui").append($banner);
        return $banner;
      }
      function showBanner(title, sub) {
        var $banner = ensureEditBanner();
        $("#sns-edit-banner-title").text(title || "\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f");
        $("#sns-edit-banner-sub").text(sub || "");
        $banner.addClass("is-open").removeClass("is-success");
      }
      function stopPolling() {
        if (editState.pollTimer) {
          clearInterval(editState.pollTimer);
          editState.pollTimer = null;
        }
      }
      function syncNativeEditText() {
        if (!editState.active || editState.apiMode) {
          return false;
        }
        var $native = nativeReply();
        var $input = uiInput();
        if (!$native.length || !$input.length) {
          return false;
        }
        var rawValue = String($native.val() || "");
        if (!rawValue) {
          return false;
        }
        editState.nativeRawLoaded = rawValue;
        var value = rawValue;
        editState.replyEncoded = "";
        var storyLineMatch = value.match(/^\s*SNSSTORY:([A-Za-z0-9+\/=]+)[ \t]*(?:\r?\n)?([\s\S]*)$/);
        if (storyLineMatch) {
          editState.storyEncoded = String(storyLineMatch[1] || "");
          value = String(storyLineMatch[2] || "");
        }
        var replyLineMatch = value.match(/^\s*SNSREPLY:([A-Za-z0-9+\/=]+)[ \t]*(?:\r?\n)?([\s\S]*)$/);
        if (replyLineMatch) {
          editState.replyEncoded = String(replyLineMatch[1] || "");
          value = String(replyLineMatch[2] || "");
        }
        var decodedAudioData = typeof window.SNSAudioDecodeMarker === "function" ? window.SNSAudioDecodeMarker(value) : null;
        var decodedVoiceText = typeof window.SNSVoiceDecodeMarker === "function" ? window.SNSVoiceDecodeMarker(value) : null;
        editState.audioMode = !!decodedAudioData;
        editState.audioData = decodedAudioData;
        editState.voiceMode = !editState.audioMode && decodedVoiceText !== null;
        var visibleValue = editState.audioMode && typeof window.SNSAudioEditText === "function" ? window.SNSAudioEditText(decodedAudioData) : editState.voiceMode ? decodedVoiceText : value;
        editState.nativeReady = true;
        $input.prop("disabled", false).val(visibleValue).attr("placeholder", "\u0418\u0437\u043c\u0435\u043d\u0438\u0442\u044c \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435...").trigger("input");
        sendButton().prop("disabled", false).html("\u2713");
        showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u0418\u0437\u043c\u0435\u043d\u0438\u0442\u0435 \u0442\u0435\u043a\u0441\u0442 \u043d\u0438\u0436\u0435 \u0438 \u043d\u0430\u0436\u043c\u0438\u0442\u0435 \u2713");
        setTimeout(function() {
          var node = $input.get(0);
          $input.focus();
          if (node) {
            try {
              node.setSelectionRange(node.value.length, node.value.length);
            } catch (error) {}
          }
        }, 20);
        return true;
      }
      function pollNativeEditText() {
        stopPolling();
        var attempts = 0;
        editState.pollTimer = setInterval(function() {
          attempts++;
          if (!editState.active || editState.apiMode) {
            stopPolling();
            return;
          }
          if (syncNativeEditText()) {
            stopPolling();
            return;
          }
          if (attempts >= 50) {
            stopPolling();
            showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043f\u043e\u043b\u0443\u0447\u0438\u0442\u044c \u0442\u0435\u043a\u0441\u0442. \u041d\u0430\u0436\u043c\u0438\u0442\u0435 \xab\u041e\u0442\u043c\u0435\u043d\u0430\xbb \u0438 \u043f\u043e\u0432\u0442\u043e\u0440\u0438\u0442\u0435.");
          }
        }, 40);
      }
      function beginVisualEdit($post, skipNativePoll) {
        if (!$post || !$post.length) {
          return;
        }
        clearNativeReleaseTimers();
        captureNativeReplyBaseline();
        stopPolling();
        editState.active = true;
        editState.saving = false;
        editState.$post = $post;
        var $send = sendButton();
        if (!editState.oldSendHtml) {
          editState.oldSendHtml = $send.html() || "\u27a4";
          editState.oldSendTitle = $send.attr("title") || "\u041e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c";
        }
        $(".sns-message").removeClass("sns-edit-selected");
        $post.addClass("sns-edit-selected");
        plusButton().prop("disabled", true).addClass("sns-edit-disabled");
        $send.html("\u2713").attr("title", "\u0421\u043e\u0445\u0440\u0430\u043d\u0438\u0442\u044c \u0438\u0437\u043c\u0435\u043d\u0435\u043d\u0438\u044f");
        uiInput().val("").attr("placeholder", "\u0417\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0442\u0435\u043a\u0441\u0442...").trigger("input");
        showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u0417\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0442\u0435\u043a\u0441\u0442...");
        if (!skipNativePoll) {
          setTimeout(pollNativeEditText, 40);
        }
      }
      function apiEditLink($post) {
        var $link = $post.find('.post-links a[href*="edit.php"]').first();
        if ($link.length) {
          return $link;
        }
        return $();
      }
      function clearApiFrameTimer() {
        if (editState.apiFrameTimer) {
          clearTimeout(editState.apiFrameTimer);
          editState.apiFrameTimer = null;
        }
      }
      function destroyApiEditFrame() {
        clearApiFrameTimer();
        if (editState.apiFrame && editState.apiFrame.length) {
          editState.apiFrame.off(".snsApiEditFrame").remove();
        }
        editState.apiFrame = null;
        editState.apiFramePhase = "";
      }
      function apiEditLoadFailure(message) {
        clearApiFrameTimer();
        editState.saving = false;
        showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", message || "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043e\u0442\u043a\u0440\u044b\u0442\u044c \u0440\u0435\u0434\u0430\u043a\u0442\u043e\u0440.");
        uiInput().prop("disabled", false).attr("placeholder", "\u0418\u0437\u043c\u0435\u043d\u0438\u0442\u044c \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435...");
        updateInlineEditSaveReady();
      }
      function clearInlineSaveTimer() {
        if (editState.inlineSaveTimer) {
          clearTimeout(editState.inlineSaveTimer);
          editState.inlineSaveTimer = null;
        }
      }
      function restoreInlineEditor() {
        if (!editState.inlineMounted || !editState.$post || !editState.$post.length) {
          editState.inlineMounted = false;
          editState.inlineOriginalHtml = "";
          return;
        }
        var $content = editState.$post.find(".post-content").first();
        if ($content.length && editState.inlineOriginalHtml !== "") {
          $content.html(editState.inlineOriginalHtml);
        }
        editState.$post.removeClass("sns-inline-edit-open");
        editState.inlineMounted = false;
        editState.inlineOriginalHtml = "";
      }
      function inlineEditValue() {
        if (!editState.$post || !editState.$post.length) {
          return "";
        }
        return String(editState.$post.find(".sns-inline-edit-textarea").first().val() || "");
      }
      function mountInlineEditor($post, value) {
        if (!$post || !$post.length) {
          return false;
        }
        var $content = $post.find(".post-content").first();
        if (!$content.length) {
          return false;
        }
        if (editState.inlineMounted) {
          var $existing = $content.find(".sns-inline-edit-textarea").first();
          if ($existing.length) {
            $existing.val(String(value || ""));
            return true;
          }
        }
        editState.inlineOriginalHtml = $content.html();
        editState.inlineMounted = true;
        var $replyPreview = $content.children(".sns-reply-preview").first().clone(true, true);
        var $editor = $('<div class="sns-inline-editor">' + '<textarea class="sns-inline-edit-textarea" spellcheck="true"></textarea>' + '<div class="sns-inline-edit-actions">' + '<button type="button" class="sns-inline-edit-cancel" title="\u041e\u0442\u043c\u0435\u043d\u0438\u0442\u044c" aria-label="\u041e\u0442\u043c\u0435\u043d\u0438\u0442\u044c">\xd7</button>' + '<button type="button" class="sns-inline-edit-save" title="\u0421\u043e\u0445\u0440\u0430\u043d\u0438\u0442\u044c" aria-label="\u0421\u043e\u0445\u0440\u0430\u043d\u0438\u0442\u044c">\u2713</button>' + "</div>" + "</div>");
        var $textarea = $editor.find(".sns-inline-edit-textarea").val(String(value || ""));
        $editor.find(".sns-inline-edit-save").prop("disabled", !editState.apiFormReady).attr("title", editState.apiFormReady ? "\u0421\u043e\u0445\u0440\u0430\u043d\u0438\u0442\u044c" : "\u041f\u043e\u0434\u0433\u043e\u0442\u0430\u0432\u043b\u0438\u0432\u0430\u044e \u0444\u043e\u0440\u043c\u0443 \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u0438\u044f...");
        $content.empty();
        if ($replyPreview.length) {
          $content.append($replyPreview);
        }
        $content.append($editor);
        $post.addClass("sns-inline-edit-open");
        function resizeInlineEditor() {
          var node = $textarea.get(0);
          if (!node) {
            return;
          }
          node.style.height = "auto";
          node.style.height = Math.min(240, Math.max(58, node.scrollHeight)) + "px";
        }
        $textarea.on("input.snsInlineEdit", resizeInlineEditor);
        $editor.on("click.snsInlineEdit", ".sns-inline-edit-save", function(event) {
          event.preventDefault();
          event.stopPropagation();
          saveApiVisualEdit();
        }).on("click.snsInlineEdit", ".sns-inline-edit-cancel", function(event) {
          event.preventDefault();
          event.stopPropagation();
          resetVisualEdit(false);
        });
        setTimeout(function() {
          resizeInlineEditor();
          $textarea.focus();
          var node = $textarea.get(0);
          if (node) {
            try {
              node.setSelectionRange(node.value.length, node.value.length);
            } catch (error) {}
          }
        }, 20);
        return true;
      }
      function renderedMessageToEditable(html) {
        var holder = document.createElement("div");
        holder.innerHTML = String(html || "");
        function walk(node) {
          if (!node) {
            return "";
          }
          if (node.nodeType === 3) {
            return String(node.nodeValue || "");
          }
          if (node.nodeType !== 1) {
            return "";
          }
          var tag = String(node.tagName || "").toLowerCase();
          if (tag === "br") {
            return "\n";
          }
          if (tag === "img") {
            var src = String(node.getAttribute("src") || "");
            return src ? "[img]" + src + "[/img]" : "";
          }
          var inner = "";
          Array.prototype.forEach.call(node.childNodes, function(child) {
            inner += walk(child);
          });
          if (tag === "b" || tag === "strong") {
            return "[b]" + inner + "[/b]";
          }
          if (tag === "i" || tag === "em") {
            return "[i]" + inner + "[/i]";
          }
          if (tag === "u") {
            return "[u]" + inner + "[/u]";
          }
          if (tag === "s" || tag === "strike" || tag === "del") {
            return "[s]" + inner + "[/s]";
          }
          if (tag === "a") {
            var href = String(node.getAttribute("href") || "");
            if (!href) {
              return inner;
            }
            if (/^\[img\][\s\S]*\[\/img\]$/i.test(inner)) {
              return inner;
            }
            var visible = String(inner || "").trim();
            if (!visible || visible === href) {
              return "[url]" + href + "[/url]";
            }
            return "[url=" + href + "]" + inner + "[/url]";
          }
          if (tag === "iframe" || tag === "video") {
            var mediaSrc = String(node.getAttribute("src") || "");
            return mediaSrc ? mediaSrc : inner;
          }
          if (tag === "p" || tag === "div" || tag === "li" || tag === "blockquote") {
            return inner + "\n";
          }
          return inner;
        }
        var result = "";
        Array.prototype.forEach.call(holder.childNodes, function(child) {
          result += walk(child);
        });
        return String(result).replace(/\u00a0/g, " ").replace(/\n{3,}/g, "\n\n").replace(/^\s*\n/, "").replace(/\n\s*$/, "");
      }
      function livePostEditableFallback($post) {
        if (!$post || !$post.length) {
          return "";
        }
        var $clone = $post.find(".post-content").first().clone();
        $clone.children(".sns-reply-preview").remove();
        return renderedMessageToEditable($clone.html());
      }
      function prepareFriendlyApiEditValue(rawSource) {
        var source = String(rawSource || "");
        var storyLineMatch = source.match(/^\s*SNSSTORY:([A-Za-z0-9+\/=]+)[ \t]*(?:\r?\n)?([\s\S]*)$/);
        if (storyLineMatch) {
          editState.storyEncoded = String(storyLineMatch[1] || editState.storyEncoded || "");
          source = String(storyLineMatch[2] || "");
        }
        var replyLineMatch = source.match(/^\s*SNSREPLY:([A-Za-z0-9+\/=]+)[ \t]*(?:\r?\n)?([\s\S]*)$/);
        if (replyLineMatch) {
          if (!editState.replyEncoded) {
            editState.replyEncoded = String(replyLineMatch[1] || "");
          }
          source = String(replyLineMatch[2] || "");
        }
        var decodedAudioData = typeof window.SNSAudioDecodeMarker === "function" ? window.SNSAudioDecodeMarker(source) : null;
        var decodedVoiceText = typeof window.SNSVoiceDecodeMarker === "function" ? window.SNSVoiceDecodeMarker(source) : null;
        editState.audioMode = !!decodedAudioData;
        editState.audioData = decodedAudioData;
        editState.voiceMode = !editState.audioMode && decodedVoiceText !== null;
        return editState.audioMode && typeof window.SNSAudioEditText === "function" ? window.SNSAudioEditText(decodedAudioData) : editState.voiceMode ? decodedVoiceText : source;
      }
      function updateInlineEditSaveReady() {
        var ready = !!(editState.apiSourceReady && editState.apiFormReady);
        sendButton().prop("disabled", !ready).html("\u2713").attr("title", ready ? "\u0421\u043e\u0445\u0440\u0430\u043d\u0438\u0442\u044c \u0438\u0437\u043c\u0435\u043d\u0435\u043d\u0438\u044f" : "\u041f\u043e\u0434\u0433\u043e\u0442\u0430\u0432\u043b\u0438\u0432\u0430\u044e \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u0438\u0435...");
      }
      function activateApiInlineSource(rawSource) {
        if (!editState.active || !editState.apiMode) {
          return;
        }
        var friendly = prepareFriendlyApiEditValue(rawSource);
        editState.apiSourceReady = true;
        editState.apiSourceValue = friendly;
        var $input = uiInput();
        $input.prop("disabled", false).val(friendly).attr("placeholder", "\u0418\u0437\u043c\u0435\u043d\u0438\u0442\u044c \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435...").trigger("input");
        updateInlineEditSaveReady();
        showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", editState.apiFormReady ? "\u0418\u0437\u043c\u0435\u043d\u0438\u0442\u0435 \u0442\u0435\u043a\u0441\u0442 \u043d\u0438\u0436\u0435 \u0438 \u043d\u0430\u0436\u043c\u0438\u0442\u0435 \u2713" : "\u0422\u0435\u043a\u0441\u0442 \u0437\u0430\u0433\u0440\u0443\u0436\u0435\u043d. \u041f\u043e\u0434\u0433\u043e\u0442\u0430\u0432\u043b\u0438\u0432\u0430\u044e \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u0438\u0435...");
        setTimeout(function() {
          var node = $input.get(0);
          $input.focus();
          if (node) {
            try {
              node.setSelectionRange(node.value.length, node.value.length);
            } catch (error) {}
          }
        }, 20);
      }
      function loadApiEditSourceText(postId) {
        var topicId = String(new URL(location.href).searchParams.get("id") || "");
        var finished = false;
        function fallback() {
          if (finished) {
            return;
          }
          finished = true;
          var raw = livePostEditableFallback(editState.$post);
          activateApiInlineSource(raw);
        }
        if (!topicId) {
          fallback();
          return;
        }
        function fetchBatch(skip) {
          if (finished || !editState.active || !editState.apiMode) {
            return;
          }
          var apiUrl = new URL("api.php", location.href);
          apiUrl.searchParams.set("method", "post.get");
          apiUrl.searchParams.set("topic_id", topicId);
          apiUrl.searchParams.set("sort_by", "id");
          apiUrl.searchParams.set("sort_dir", "desc");
          apiUrl.searchParams.set("limit", "100");
          apiUrl.searchParams.set("skip", String(skip));
          apiUrl.searchParams.set("fields", "id,message,topic_id");
          apiUrl.searchParams.set("_sns_edit_source", String(Date.now()));
          SNSRequest({
            url: apiUrl.toString(),
            type: "GET",
            dataType: "json",
            cache: false,
            timeout: 6500
          }).done(function(json) {
            if (finished) {
              return;
            }
            var rows = json && Array.isArray(json.response) ? json.response : [];
            var found = null;
            rows.some(function(row) {
              if (String(row && row.id || "") === String(postId)) {
                found = row;
                return true;
              }
              return false;
            });
            if (found) {
              finished = true;
              activateApiInlineSource(renderedMessageToEditable(found.message));
              return;
            }
            if (rows.length >= 100 && skip < 900) {
              fetchBatch(skip + 100);
              return;
            }
            fallback();
          }).fail(fallback);
        }
        fetchBatch(0);
      }
      function waitForApiEditFrameReady(attempt) {
        attempt = attempt || 0;
        if (!editState.active || !editState.apiMode || editState.apiFramePhase !== "loading") {
          return;
        }
        if (hydrateApiEditFromFrame()) {
          clearApiFrameTimer();
          editState.apiFramePhase = "ready";
          return;
        }
        if (attempt >= 80) {
          editState.apiFormReady = false;
          updateInlineEditSaveReady();
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u0422\u0435\u043a\u0441\u0442 \u043c\u043e\u0436\u043d\u043e \u0438\u0437\u043c\u0435\u043d\u0438\u0442\u044c, \u043d\u043e \u0444\u043e\u0440\u043c\u0430 \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u0438\u044f \u0435\u0449\u0435 \u043d\u0435 \u0433\u043e\u0442\u043e\u0432\u0430.");
          return;
        }
        editState.apiFrameTimer = setTimeout(function() {
          waitForApiEditFrameReady(attempt + 1);
        }, 60);
      }
      function hydrateApiEditFromFrame() {
        if (!editState.active || !editState.apiMode || !editState.apiFrame || !editState.apiFrame.length) {
          return false;
        }
        var frameNode = editState.apiFrame.get(0);
        var frameDoc;
        try {
          frameDoc = frameNode.contentDocument || frameNode.contentWindow && frameNode.contentWindow.document;
        } catch (error) {
          return false;
        }
        if (!frameDoc) {
          return false;
        }
        var $frameDoc = $(frameDoc);
        var $form = $frameDoc.find("form#post").first();
        if (!$form.length) {
          $form = $frameDoc.find("form").filter(function() {
            return $(this).find('#main-reply, textarea[name="req_message"], textarea').length > 0;
          }).first();
        }
        var $textarea = $form.find('#main-reply, textarea[name="req_message"], textarea').first();
        if (!$form.length || !$textarea.length) {
          return false;
        }
        editState.apiForm = $form;
        editState.apiTextarea = $textarea;
        var $submit = $form.find('input[type="submit"], button[type="submit"]').filter(function() {
          var label = cleanText($(this).val() || $(this).text() || "");
          return !/\u043f\u0440\u0435\u0434\u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440|preview/i.test(label);
        }).first();
        editState.apiSubmitButton = $submit;
        editState.apiSubmitName = $submit.length ? String($submit.attr("name") || "") : "";
        editState.apiSubmitValue = $submit.length ? String($submit.val() || $submit.text() || "") : "";
        editState.apiFormReady = true;
        updateInlineEditSaveReady();
        if (editState.apiSourceReady) {
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u0418\u0437\u043c\u0435\u043d\u0438\u0442\u0435 \u0442\u0435\u043a\u0441\u0442 \u043d\u0438\u0436\u0435 \u0438 \u043d\u0430\u0436\u043c\u0438\u0442\u0435 \u2713");
        }
        return true;
      }
      function serverMessageToFriendlyEditValue(message) {
        var source = renderedMessageToEditable(message);
        var storyLineMatch = source.match(/^\s*SNSSTORY:([A-Za-z0-9+\/=]+)[ \t]*(?:\r?\n)?([\s\S]*)$/);
        if (storyLineMatch) {
          source = String(storyLineMatch[2] || "");
        }
        var replyLineMatch = source.match(/^\s*SNSREPLY:([A-Za-z0-9+\/=]+)[ \t]*(?:\r?\n)?([\s\S]*)$/);
        if (replyLineMatch) {
          source = String(replyLineMatch[2] || "");
        }
        var decodedAudioData = typeof window.SNSAudioDecodeMarker === "function" ? window.SNSAudioDecodeMarker(source) : null;
        if (decodedAudioData && typeof window.SNSAudioEditText === "function") {
          return window.SNSAudioEditText(decodedAudioData);
        }
        var decodedVoiceText = typeof window.SNSVoiceDecodeMarker === "function" ? window.SNSVoiceDecodeMarker(source) : null;
        if (decodedVoiceText !== null) {
          return decodedVoiceText;
        }
        return source;
      }
      function normalizeEditedCompare(value) {
        return String(value || "").replace(/\r\n?/g, "\n").replace(/[ \t]+\n/g, "\n").trim();
      }
      function refreshEditedApiPost(postId, expectedVisible, attempt) {
        attempt = attempt || 0;
        var topicId = String(new URL(location.href).searchParams.get("id") || "");
        if (!topicId) {
          editState.saving = false;
          uiInput().prop("disabled", false);
          updateInlineEditSaveReady();
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043f\u0440\u043e\u0432\u0435\u0440\u0438\u0442\u044c \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u0438\u0435.");
          return;
        }
        var completed = false;
        function finishNotSaved(latestRow) {
          if (completed) {
            return;
          }
          completed = true;
          editState.saving = false;
          uiInput().prop("disabled", false);
          updateInlineEditSaveReady();
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u0424\u043e\u0440\u0443\u043c \u043d\u0435 \u043f\u043e\u0434\u0442\u0432\u0435\u0440\u0434\u0438\u043b \u0438\u0437\u043c\u0435\u043d\u0435\u043d\u0438\u0435. \u0422\u0435\u043a\u0441\u0442 \u043e\u0441\u0442\u0430\u043b\u0441\u044f \u0432 \u0440\u0435\u0434\u0430\u043a\u0442\u043e\u0440\u0435.");
        }
        function applySavedRow(row) {
          if (completed) {
            return;
          }
          completed = true;
          clearInlineSaveTimer();
          var $livePost = editState.$post && editState.$post.length ? editState.$post : $("#p" + postId);
          var $liveContent = $livePost.find(".post-content").first();
          if (row && typeof row.message === "string" && $liveContent.length) {
            $liveContent.html(row.message);
            $livePost.removeAttr("data-sns-reply data-sns-story data-sns-voice data-sns-audio data-sns-images").removeClass("sns-has-reply sns-voice-message sns-voice-expanded sns-audio-message sns-has-media sns-media-only sns-media-caption sns-image-only");
            if (typeof window.SNSEnhancePosts === "function") {
              window.SNSEnhancePosts($livePost);
            }
            $livePost.addClass("sns-edit-updated-flash");
            setTimeout(function() {
              $livePost.removeClass("sns-edit-updated-flash");
            }, 850);
          }
          if (editState.active && !editState.apiMode) {
            releaseNativeQuickEditMode();
          }
          resetVisualEdit(true);
        }
        function fetchBatch(skip) {
          if (completed || !editState.active) {
            return;
          }
          var apiUrl = new URL("api.php", location.href);
          apiUrl.searchParams.set("method", "post.get");
          apiUrl.searchParams.set("topic_id", topicId);
          apiUrl.searchParams.set("sort_by", "id");
          apiUrl.searchParams.set("sort_dir", "desc");
          apiUrl.searchParams.set("limit", "100");
          apiUrl.searchParams.set("skip", String(skip));
          apiUrl.searchParams.set("fields", "id,message,topic_id,edited");
          apiUrl.searchParams.set("_sns_edit_verify", String(Date.now()));
          SNSRequest({
            url: apiUrl.toString(),
            type: "GET",
            dataType: "json",
            cache: false,
            timeout: 6500
          }).done(function(json) {
            if (completed) {
              return;
            }
            var rows = json && Array.isArray(json.response) ? json.response : [];
            var found = null;
            rows.some(function(row) {
              if (String(row && row.id || "") === String(postId)) {
                found = row;
                return true;
              }
              return false;
            });
            if (found) {
              var serverFriendly = serverMessageToFriendlyEditValue(found.message);
              var expected = normalizeEditedCompare(expectedVisible);
              var actual = normalizeEditedCompare(serverFriendly);
              if (actual === expected) {
                applySavedRow(found);
                return;
              }
              if (attempt < 8) {
                setTimeout(function() {
                  refreshEditedApiPost(postId, expectedVisible, attempt + 1);
                }, 450);
                return;
              }
              finishNotSaved(found);
              return;
            }
            if (rows.length >= 100 && skip < 900) {
              fetchBatch(skip + 100);
              return;
            }
            if (attempt < 8) {
              setTimeout(function() {
                refreshEditedApiPost(postId, expectedVisible, attempt + 1);
              }, 450);
              return;
            }
            finishNotSaved(null);
          }).fail(function() {
            if (attempt < 8) {
              setTimeout(function() {
                refreshEditedApiPost(postId, expectedVisible, attempt + 1);
              }, 500);
              return;
            }
            finishNotSaved(null);
          });
        }
        fetchBatch(0);
      }
      function finishApiFrameSave() {
        clearApiFrameTimer();
        if (!editState.active || !editState.apiMode) {
          return;
        }
        refreshEditedApiPost(editState.apiPostId, editState.apiSourceValue);
      }
      function beginApiVisualEdit($post) {
        var $link = apiEditLink($post);
        if (!$link.length) {
          return;
        }
        if (editState.active) {
          resetVisualEdit(false);
        }
        beginVisualEdit($post, true);
        editState.apiMode = true;
        uiInput().prop("disabled", true).val("").attr("placeholder", "\u0417\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0442\u0435\u043a\u0441\u0442...");
        sendButton().prop("disabled", true).html("\u2713").attr("title", "\u041f\u043e\u0434\u0433\u043e\u0442\u0430\u0432\u043b\u0438\u0432\u0430\u044e \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u0438\u0435...");
        editState.apiPostId = String($post.attr("id") || "").replace(/^p/i, "");
        editState.apiEditUrl = new URL($link.attr("href"), location.href).toString();
        editState.apiForm = null;
        editState.apiTextarea = null;
        editState.apiSubmitButton = null;
        editState.apiFormReady = false;
        editState.apiSourceReady = false;
        editState.apiSourceValue = "";
        editState.apiSubmitName = "";
        editState.apiSubmitValue = "";
        editState.replyEncoded = String($post.attr("data-sns-reply") || "");
        editState.storyEncoded = String($post.attr("data-sns-story") || "");
        destroyApiEditFrame();
        loadApiEditSourceText(editState.apiPostId);
        var frameName = "sns-edit-frame-" + Date.now();
        var $frame = $("<iframe " + 'class="sns-api-edit-frame" ' + 'aria-hidden="true" ' + 'tabindex="-1"></iframe>').attr("name", frameName).css({
          position: "fixed",
          left: "-10000px",
          top: "-10000px",
          width: "1px",
          height: "1px",
          opacity: 0,
          border: 0,
          pointerEvents: "none"
        });
        editState.apiFrame = $frame;
        editState.apiFramePhase = "loading";
        $frame.on("load.snsApiEditFrame", function() {
          if (!editState.active || !editState.apiMode) {
            return;
          }
          if (editState.apiFramePhase !== "loading") {
            return;
          }
          clearApiFrameTimer();
          waitForApiEditFrameReady(0);
        });
        $frame.attr("src", editState.apiEditUrl);
        $("body").append($frame);
        editState.apiFrameTimer = setTimeout(function() {
          if (editState.active && editState.apiMode && editState.apiFramePhase === "loading") {
            apiEditLoadFailure("\u0424\u043e\u0440\u043c\u0430 \u0440\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u044f \u043d\u0435 \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u043b\u0430\u0441\u044c.");
          }
        }, 12e3);
      }
      function serializeRusffEditForm(formNode) {
        if (!formNode) {
          return null;
        }
        var frameNode = editState.apiFrame && editState.apiFrame.length ? editState.apiFrame.get(0) : null;
        var frameWindow = frameNode && frameNode.contentWindow ? frameNode.contentWindow : null;
        var candidates = [];
        if (frameWindow && frameWindow.jQuery) {
          candidates.push(frameWindow.jQuery);
        }
        if (window.jQuery && candidates.indexOf(window.jQuery) === -1) {
          candidates.push(window.jQuery);
        }
        for (var i = 0; i < candidates.length; i++) {
          var jq = candidates[i];
          var $form = jq(formNode);
          try {
            if (typeof $form.serialize2 === "function") {
              var methodResult = $form.serialize2();
              if (methodResult !== undefined && methodResult !== null) {
                return {
                  data: methodResult,
                  jq: jq,
                  mode: "fn.serialize2"
                };
              }
            }
          } catch (methodError) {}
          if (typeof jq.serialize2 === "function") {
            var attempts = [ function() {
              return jq.serialize2(formNode);
            }, function() {
              return jq.serialize2($form);
            }, function() {
              return jq.serialize2($form.serializeArray());
            } ];
            for (var j = 0; j < attempts.length; j++) {
              try {
                var result = attempts[j]();
                if (result !== undefined && result !== null) {
                  return {
                    data: result,
                    jq: jq,
                    mode: "$.serialize2"
                  };
                }
              } catch (serializeError) {}
            }
          }
        }
        return null;
      }
      function saveApiVisualEdit() {
        if (!editState.active || !editState.apiMode || editState.saving || !editState.apiFormReady || !editState.apiForm || !editState.apiTextarea) {
          return;
        }
        var visibleValue = String(uiInput().val() || "");
        var value = visibleValue;
        if (editState.audioMode && typeof window.SNSAudioParseEditText === "function" && typeof window.SNSAudioEncodeMarker === "function") {
          var parsedAudioEdit = window.SNSAudioParseEditText(visibleValue);
          if (parsedAudioEdit) {
            value = window.SNSAudioEncodeMarker(parsedAudioEdit);
          }
        } else if (editState.voiceMode && typeof window.SNSVoiceEncodeMarker === "function") {
          value = window.SNSVoiceEncodeMarker(visibleValue);
        }
        if (editState.replyEncoded) {
          value = "SNSREPLY:" + editState.replyEncoded + "\n" + value;
        }
        if (editState.storyEncoded) {
          value = "SNSSTORY:" + editState.storyEncoded + "\n" + value;
        }
        editState.saving = true;
        editState.apiSourceValue = visibleValue;
        uiInput().prop("disabled", true);
        sendButton().prop("disabled", true);
        showBanner("\u0421\u043e\u0445\u0440\u0430\u043d\u044f\u044e \u0438\u0437\u043c\u0435\u043d\u0435\u043d\u0438\u044f...", "");
        editState.apiTextarea.val(value);
        var formNode = editState.apiForm.get(0);
        if (!formNode) {
          editState.saving = false;
          uiInput().prop("disabled", false);
          updateInlineEditSaveReady();
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u0424\u043e\u0440\u043c\u0430 \u0440\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u044f \u043f\u043e\u0442\u0435\u0440\u044f\u043d\u0430.");
          return;
        }
        editState.apiForm.find("input.sns-api-submit-proxy").remove();
        if (editState.apiSubmitName) {
          $('<input type="hidden" class="sns-api-submit-proxy">').attr("name", editState.apiSubmitName).val(editState.apiSubmitValue).appendTo(editState.apiForm);
        }
        var serialized = serializeRusffEditForm(formNode);
        if (!serialized) {
          editState.saving = false;
          uiInput().prop("disabled", false);
          updateInlineEditSaveReady();
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u041d\u0430 \u044d\u0442\u043e\u043c \u0444\u043e\u0440\u0443\u043c\u0435 \u043d\u0435 \u043d\u0430\u0448\u043b\u0430\u0441\u044c RusFF $.serialize2. \u0422\u0435\u043a\u0441\u0442 \u043e\u0441\u0442\u0430\u043b\u0441\u044f \u0432 \u0440\u0435\u0434\u0430\u043a\u0442\u043e\u0440\u0435.");
          return;
        }
        var action = editState.apiForm.attr("action") || editState.apiEditUrl;
        action = new URL(action, editState.apiEditUrl || location.href).toString();
        var method = String(editState.apiForm.attr("method") || "POST").toUpperCase();
        clearApiFrameTimer();
        clearInlineSaveTimer();
        SNSRequest({
          url: action,
          type: method,
          data: serialized.data,
          cache: false,
          timeout: 12e3
        }, serialized.jq).done(function() {
          if (!editState.active || !editState.apiMode) {
            return;
          }
          setTimeout(function() {
            refreshEditedApiPost(editState.apiPostId, visibleValue, 0);
          }, 250);
        }).fail(function(xhr, status, error) {
          try {
            console.error("[SNS edit serialize2]", serialized.mode, status, error, xhr && xhr.status);
          } catch (consoleError) {}
          editState.saving = false;
          uiInput().prop("disabled", false);
          updateInlineEditSaveReady();
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c \u0440\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0447\u0435\u0440\u0435\u0437 RusFF AJAX. \u0422\u0435\u043a\u0441\u0442 \u043e\u0441\u0442\u0430\u043b\u0441\u044f \u0432 \u0440\u0435\u0434\u0430\u043a\u0442\u043e\u0440\u0435.");
        });
      }
      function findNativeCancel() {
        var $form = $("#post").first();
        var $result = $();
        $form.find("input, button, a").each(function() {
          var $control = $(this);
          var label = cleanText($control.val() || $control.text() || $control.attr("title") || "");
          var name = String($control.attr("name") || "");
          if (/\u043e\u0442\u043c\u0435\u043d|cancel/i.test(label) || /cancel/i.test(name)) {
            $result = $control;
            return false;
          }
        });
        return $result;
      }
      function resetVisualEdit(showSuccess) {
        stopPolling();
        clearInlineSaveTimer();
        destroyApiEditFrame();
        if (editState.inlineMounted) {
          restoreInlineEditor();
        }
        var $input = uiInput();
        var $send = sendButton();
        $(".sns-message").removeClass("sns-edit-selected");
        $input.val("").prop("disabled", false).css("height", "40px").attr("placeholder", "\u041d\u0430\u043f\u0438\u0441\u0430\u0442\u044c \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435...").trigger("input");
        nativeReply().val("");
        plusButton().prop("disabled", false).removeClass("sns-edit-disabled");
        $send.prop("disabled", false).html(editState.oldSendHtml || "\u27a4").attr("title", editState.oldSendTitle || "\u041e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c");
        editState.active = false;
        editState.saving = false;
        editState.$post = $();
        editState.apiMode = false;
        editState.apiPostId = "";
        editState.apiEditUrl = "";
        editState.apiForm = null;
        editState.apiTextarea = null;
        editState.apiSubmitName = "";
        editState.apiSubmitValue = "";
        editState.voiceMode = false;
        editState.audioMode = false;
        editState.audioData = null;
        editState.replyMeta = null;
        editState.replyEncoded = "";
        editState.storyEncoded = "";
        editState.apiFrame = null;
        editState.apiFrameTimer = null;
        editState.apiFramePhase = "";
        editState.apiSubmitButton = null;
        editState.apiFormReady = false;
        editState.apiSourceReady = false;
        editState.apiSourceValue = "";
        editState.inlineMounted = false;
        editState.inlineOriginalHtml = "";
        editState.inlineSaveTimer = null;
        editState.nativeReady = false;
        editState.nativeRawLoaded = "";
        if (editState.nativeProxyRestoreTimer) {
          clearTimeout(editState.nativeProxyRestoreTimer);
          editState.nativeProxyRestoreTimer = null;
        }
        if (showSuccess) {
          showBanner("\u2713 \u0421\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435 \u0438\u0437\u043c\u0435\u043d\u0435\u043d\u043e", "");
          $("#sns-edit-banner").addClass("is-success");
          $("#sns-edit-cancel").hide();
          setTimeout(function() {
            $("#sns-edit-banner").removeClass("is-open is-success");
            $("#sns-edit-cancel").show();
          }, 1200);
        } else {
          $("#sns-edit-banner").removeClass("is-open is-success");
        }
      }
      function saveThroughNativeQuickEdit() {
        if (!editState.active || editState.apiMode || editState.saving) {
          return;
        }
        if (!editState.nativeReady) {
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u0420\u0435\u0436\u0438\u043c \u0440\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u044f RusFF \u0435\u0449\u0435 \u043d\u0435 \u0433\u043e\u0442\u043e\u0432.");
          return;
        }
        var $native = nativeReply();
        var $submit = nativeQuickEditSubmitButton();
        if (!$native.length || !$submit.length) {
          showBanner("\u0420\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f", "\u0420\u043e\u0434\u043d\u0430\u044f \u0444\u043e\u0440\u043c\u0430 RusFF \u043d\u0435 \u0433\u043e\u0442\u043e\u0432\u0430 \u043a \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u0438\u044e.");
          return;
        }
        var visibleValue = String(uiInput().val() || "");
        var rawValue = visibleValue;
        if (editState.audioMode && typeof window.SNSAudioParseEditText === "function" && typeof window.SNSAudioEncodeMarker === "function") {
          var parsedAudio = window.SNSAudioParseEditText(visibleValue);
          if (parsedAudio) {
            rawValue = window.SNSAudioEncodeMarker(parsedAudio);
          }
        } else if (editState.voiceMode && typeof window.SNSVoiceEncodeMarker === "function") {
          rawValue = window.SNSVoiceEncodeMarker(visibleValue);
        }
        if (editState.replyEncoded) {
          rawValue = "SNSREPLY:" + editState.replyEncoded + "\n" + rawValue;
        }
        if (editState.storyEncoded) {
          rawValue = "SNSSTORY:" + editState.storyEncoded + "\n" + rawValue;
        }
        editState.saving = true;
        uiInput().prop("disabled", true);
        sendButton().prop("disabled", true);
        showBanner("\u0421\u043e\u0445\u0440\u0430\u043d\u044f\u044e \u0438\u0437\u043c\u0435\u043d\u0435\u043d\u0438\u044f...", "");
        $native.val(rawValue).trigger("input").trigger("change");
        $submit.trigger("click");
      }
      document.addEventListener("click", function(event) {
        var target = event.target;
        if (!target || !target.closest) {
          return;
        }
        var editButton = target.closest(".sns-action-edit");
        if (!editButton) {
          return;
        }
        var post = editButton.closest(".post");
        if (!post) {
          return;
        }
        var $post = $(post);
        if ($post.attr("data-sns-api-post") === "1") {
          event.preventDefault();
          event.stopPropagation();
          event.stopImmediatePropagation();
          launchNativeQuickEditForPost($post);
          return;
        }
        launchNativeQuickEditForPost($post);
      }, true);
      document.addEventListener("click", function(event) {
        if (!editState.active || editState.apiMode) {
          return;
        }
        var target = event.target;
        if (!target || !target.closest) {
          return;
        }
        var send = target.closest(".sns-ui-send");
        if (!send) {
          return;
        }
        event.preventDefault();
        event.stopPropagation();
        event.stopImmediatePropagation();
        saveThroughNativeQuickEdit();
      }, true);
      document.addEventListener("keydown", function(event) {
        if (!editState.active || editState.apiMode) {
          return;
        }
        var target = event.target;
        if (!target || target.id !== "sns-ui-input") {
          return;
        }
        if (event.key === "Enter" && (event.ctrlKey || event.metaKey)) {
          event.preventDefault();
          event.stopPropagation();
          event.stopImmediatePropagation();
          saveThroughNativeQuickEdit();
        }
      }, true);
      $(document).off("pun_preedit.snsClearEdit").on("pun_preedit.snsClearEdit", function() {
        if (!editState.active) {
          return;
        }
        setTimeout(pollNativeEditText, 20);
      });
      $(document).off("click.snsClearEditSaving", ".sns-ui-send").on("click.snsClearEditSaving", ".sns-ui-send", function() {
        if (editState.active && !editState.apiMode) {
          editState.saving = true;
          showBanner("\u0421\u043e\u0445\u0440\u0430\u043d\u044f\u044e \u0438\u0437\u043c\u0435\u043d\u0435\u043d\u0438\u044f...", "");
        }
      });
      function refreshNativeEditedBubble() {
        if (!editState.active || !editState.$post || !editState.$post.length) {
          resetVisualEdit(true);
          return;
        }
        var postId = String(editState.$post.attr("id") || "").replace(/^p/i, "").replace(/\D+/g, "");
        var expectedVisible = String(uiInput().val() || "");
        if (!postId) {
          resetVisualEdit(true);
          return;
        }
        refreshEditedApiPost(postId, expectedVisible, 0);
      }
      $(document).off("pun_edit.snsClearEdit").on("pun_edit.snsClearEdit", function() {
        if (!editState.active) {
          return;
        }
        releaseNativeQuickEditMode();
        setTimeout(function() {
          refreshNativeEditedBubble();
        }, 40);
      });
      $(document).off("click.snsClearEditCancel", "#sns-edit-cancel").on("click.snsClearEditCancel", "#sns-edit-cancel", function(event) {
        event.preventDefault();
        if (!editState.active) {
          resetVisualEdit(false);
          return;
        }
        var $cancel = findNativeCancel();
        if ($cancel.length) {
          try {
            $cancel.get(0).click();
          } catch (error) {}
          resetVisualEdit(false);
          return;
        }
      });
      var style = document.createElement("style");
      style.id = "sns-clear-edit-mode-style";
      style.textContent = [ "body.sns-chat-page #sns-composer-ui{position:relative!important;}", "body.sns-chat-page #sns-edit-banner{", "position:absolute;", "left:16px;right:16px;bottom:calc(100% + 8px);", "z-index:1000;", "display:none;", "box-sizing:border-box;", "min-height:42px;", "padding:8px 10px 8px 12px;", "align-items:center;", "justify-content:space-between;", "gap:12px;", "background:rgba(255,255,255,.97);", "border:1px solid rgba(0,0,0,.08);", "border-radius:10px;", "box-shadow:0 8px 24px rgba(0,0,0,.12);", "}", "body.sns-chat-page #sns-edit-banner.is-open{display:flex;}", "body.sns-chat-page #sns-edit-banner.is-success{", "background:#f5f1e9;", "}", "body.sns-chat-page .sns-edit-banner-main{", "display:flex;align-items:center;gap:9px;min-width:0;", "}", "body.sns-chat-page .sns-edit-pencil{", "font-size:15px;color:var(--sns-own-g1,#b65f3a);flex:0 0 auto;", "}", "body.sns-chat-page .sns-edit-banner-copy{", "display:flex;flex-direction:column;gap:2px;min-width:0;", "}", "body.sns-chat-page #sns-edit-banner-title{", "font:600 11px/1.2 Arial,sans-serif;color:#333;", "}", "body.sns-chat-page #sns-edit-banner-sub{", "font:9px/1.25 Arial,sans-serif;color:#888;", "white-space:nowrap;overflow:hidden;text-overflow:ellipsis;", "}", "body.sns-chat-page #sns-edit-cancel{", "flex:0 0 auto;", "padding:6px 9px;", "cursor:pointer;", "background:transparent;", "color:#777;", "border:0;", "border-radius:7px;", "font:10px Arial,sans-serif;", "}", "body.sns-chat-page #sns-edit-cancel:hover{", "background:rgba(0,0,0,.05);color:#333;", "}", "body.sns-chat-page .sns-message.sns-edit-selected .post-content{", "outline:2px solid var(--sns-own-g1,#b65f3a)!important;", "outline-offset:3px!important;", "}", "body.sns-chat-page .sns-message.sns-edit-updated-flash .post-content{animation:snsEditUpdatedFlash .8s ease!important;}", "@keyframes snsEditUpdatedFlash{0%{filter:brightness(1.22);}100%{filter:brightness(1);}}", "body.sns-chat-page .sns-ui-plus.sns-edit-disabled{", "opacity:.28!important;cursor:default!important;", "}", "body.sns-chat-page .sns-message.sns-edit-selected{", "position:relative;", "}", "@media(max-width:650px){", "body.sns-chat-page #sns-edit-banner{left:8px;right:8px;}", "body.sns-chat-page #sns-edit-banner-sub{max-width:180px;}", "}" ].join("");
      document.head.appendChild(style);
    })(jQuery);
    (function($) {
      "use strict";
      var MIN_H = 42;
      var AUTO_MAX_H = 220;
      var MANUAL_MAX_H = 380;
      var manualHeight = MIN_H;
      var dragging = false;
      var startY = 0;
      var startHeight = MIN_H;
      function $composer() {
        return $("#sns-composer-ui").first();
      }
      function $wrap() {
        return $("#sns-composer-ui>.sns-ui-input-wrap").first();
      }
      function $input() {
        return $("#sns-ui-input").first();
      }
      function clampHeight(value) {
        value = parseFloat(value);
        if (!isFinite(value)) {
          value = MIN_H;
        }
        return Math.max(MIN_H, Math.min(MANUAL_MAX_H, value));
      }
      function setRealHeight(height) {
        height = clampHeight(height);
        var wrap = $wrap().get(0);
        var input = $input().get(0);
        if (wrap) {
          wrap.style.setProperty("height", Math.round(height) + "px", "important");
          wrap.style.setProperty("min-height", Math.round(height) + "px", "important");
          wrap.style.setProperty("max-height", Math.round(height) + "px", "important");
        }
        if (input) {
          input.style.setProperty("height", Math.round(height) + "px", "important");
          input.style.setProperty("min-height", Math.round(height) + "px", "important");
          input.style.setProperty("max-height", Math.round(height) + "px", "important");
        }
      }
      function measuredContentHeight() {
        var input = $input().get(0);
        if (!input) {
          return MIN_H;
        }
        var oldHeight = input.style.getPropertyValue("height");
        var oldPriority = input.style.getPropertyPriority("height");
        input.style.setProperty("height", MIN_H + "px", "important");
        var needed = Math.max(MIN_H, input.scrollHeight);
        if (oldHeight) {
          input.style.setProperty("height", oldHeight, oldPriority || "important");
        } else {
          input.style.removeProperty("height");
        }
        return Math.min(needed, MANUAL_MAX_H);
      }
      function updateOverflow(height) {
        var input = $input().get(0);
        if (!input) {
          return;
        }
        input.style.setProperty("overflow-y", input.scrollHeight > height ? "auto" : "hidden", "important");
      }
      function syncManualHeight() {
        var target = clampHeight(manualHeight);
        setRealHeight(target);
        var input = $input().get(0);
        if (input) {
          input.style.setProperty("overflow-y", input.scrollHeight > target ? "auto" : "hidden", "important");
        }
      }
      function resetHeight() {
        manualHeight = MIN_H;
        setRealHeight(MIN_H);
        updateOverflow(MIN_H);
      }
      function currentHeight() {
        var input = $input().get(0);
        if (!input) {
          return MIN_H;
        }
        return clampHeight(input.getBoundingClientRect().height);
      }
      function ensureHandle() {
        var composer = $composer();
        if (!composer.length) {
          return $();
        }
        var handle = composer.children(".sns-composer-resize-handle-v38");
        if (!handle.length) {
          handle = $("<button " + 'type="button" ' + 'class="sns-composer-resize-handle-v38" ' + 'title="\u0418\u0437\u043c\u0435\u043d\u0438\u0442\u044c \u0432\u044b\u0441\u043e\u0442\u0443 \u043f\u043e\u043b\u044f \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f">' + "<span></span>" + "</button>");
          composer.append(handle);
        }
        return handle;
      }
      function bindResize() {
        var handle = ensureHandle();
        if (!handle.length) {
          return;
        }
        handle.off(".snsComposerV38").on("mousedown.snsComposerV38", function(event) {
          if (event.which !== 1) {
            return;
          }
          event.preventDefault();
          event.stopPropagation();
          dragging = true;
          startY = event.clientY;
          startHeight = currentHeight();
          manualHeight = startHeight;
          $("body").addClass("sns-composer-v38-dragging");
        }).on("dblclick.snsComposerV38", function(event) {
          event.preventDefault();
          resetHeight();
        });
        $(document).off("mousemove.snsComposerV38 mouseup.snsComposerV38").on("mousemove.snsComposerV38", function(event) {
          if (!dragging) {
            return;
          }
          event.preventDefault();
          var delta = event.clientY - startY;
          manualHeight = clampHeight(startHeight + delta);
          setRealHeight(manualHeight);
          updateOverflow(manualHeight);
        }).on("mouseup.snsComposerV38", function() {
          if (!dragging) {
            return;
          }
          dragging = false;
          $("body").removeClass("sns-composer-v38-dragging");
          syncManualHeight();
        });
      }
      function bindManualOverflow() {
        $input().off("input.snsComposer");
        $(document).off("input.snsComposerManualV39", "#sns-ui-input").on("input.snsComposerManualV39", "#sns-ui-input", function() {
          syncManualHeight();
        });
        $(document).off("pun_post.snsComposerManualV39").on("pun_post.snsComposerManualV39", function() {
          setTimeout(function() {
            if (!$.trim($input().val())) {
              resetHeight();
            }
          }, 30);
        });
        $(document).off("pun_edit.snsComposerManualV39").on("pun_edit.snsComposerManualV39", function() {
          setTimeout(resetHeight, 90);
        });
      }
      function apiHistoryUrl(method) {
        var url = new URL("api.php", location.href);
        url.searchParams.set("method", method);
        return url;
      }
      function fetchTopicPostCount(topicId) {
        var url = apiHistoryUrl("topic.get");
        url.searchParams.set("topic_id", topicId);
        url.searchParams.set("fields", "id,num_replies");
        url.searchParams.set("_sns_history_count", String(Date.now()));
        return SNSRequest({
          url: url.toString(),
          type: "GET",
          dataType: "json",
          cache: false,
          timeout: 8e3
        }).then(function(json) {
          var topics = json && Array.isArray(json.response) ? json.response : [];
          if (!topics.length) {
            return 0;
          }
          var replies = parseInt(topics[0].num_replies, 10);
          if (!isFinite(replies) || replies < 0) {
            return 0;
          }
          return replies + 1;
        }).catch(function() {
          return 0;
        });
      }
      function fetchHistoryApiBatch(topicId, skip) {
        var url = apiHistoryUrl("post.get");
        url.searchParams.set("topic_id", topicId);
        url.searchParams.set("sort_by", "id");
        url.searchParams.set("sort_dir", "asc");
        url.searchParams.set("skip", String(skip));
        url.searchParams.set("limit", "100");
        url.searchParams.set("fields", "id,username,user_id,message,posted,topic_id,avatar");
        url.searchParams.set("_sns_full_history", String(Date.now()) + "_" + skip);
        return SNSRequest({
          url: url.toString(),
          type: "GET",
          dataType: "json",
          cache: false,
          timeout: 1e4
        }).then(function(json) {
          return json && Array.isArray(json.response) ? json.response.slice() : [];
        });
      }
      function appendApiHistoryBatch($topic, posts) {
        var added = 0;
        var raw = 0;
        var configs = 0;
        posts.forEach(function(post) {
          if (!post || !post.id) {
            return;
          }
          raw++;
          if (apiMessageIsConfig(post.message)) {
            configs++;
            return;
          }
          if (apiPostExists(post.id)) {
            return;
          }
          var reservedId = String(post.id);
          apiPostReserved[reservedId] = true;
          var $newPost = createApiPostNode(post);
          $newPost.addClass("sns-raw-awaiting-enhance");
          if (post.avatar && $newPost.length) {
            var $list = $newPost.find(".post-author ul").first();
            if ($list.length) {
              $('<li class="pa-avatar"></li>').append($('<img alt="">').attr("src", String(post.avatar))).appendTo($list);
            }
          }
          $topic.append($newPost);
          delete apiPostReserved[reservedId];
          added++;
        });
        return {
          raw: raw,
          configs: configs,
          added: added
        };
      }
      function syncAllPages() {
        var topicIdValue = apiTopicId();
        var $topic = $("#sns-chat-shell>.topic").first();
        if (!topicIdValue || !$topic.length) {
          return;
        }
        var $loading = $("#sns-pages-loading");
        if (!$loading.length) {
          $loading = $('<div id="sns-pages-loading">' + "\u0437\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0432\u0441\u044e \u0438\u0441\u0442\u043e\u0440\u0438\u044e\u2026" + "</div>");
          $topic.append($loading);
        }
        var expectedRaw = 0;
        var fetchedRaw = 0;
        var fetchedConfigs = 0;
        var addedReal = 0;
        var skip = 0;
        var failed = false;
        var MAX_SKIP = 1e3;
        function updateStatus() {
          var target = expectedRaw > 0 ? " / " + expectedRaw : "";
          $loading.text("\u0437\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0438\u0441\u0442\u043e\u0440\u0438\u044e: " + fetchedRaw + target);
        }
        function finalizeApiHistory() {
          sortTopicPostsChronologically($topic);
          window.__SNS_FULL_HISTORY_EXPECTED_RAW__ = expectedRaw;
          window.__SNS_FULL_HISTORY_FETCHED_RAW__ = fetchedRaw;
          window.__SNS_FULL_HISTORY_CONFIG_POSTS__ = fetchedConfigs;
          window.__SNS_FULL_HISTORY_REAL_POSTS__ = Math.max(0, fetchedRaw - fetchedConfigs - 1);
          window.__SNS_FULL_HISTORY_ADDED__ = addedReal;
          $("#sns-pages-loading").stop(true, true).remove();
          $("#sns-empty").remove();
          $(document).trigger("sns_pages_loaded");
          if (!window.__SNS_INITIAL_HISTORY_READY__) {
            window.__SNS_INITIAL_HISTORY_READY__ = true;
            $(document).trigger("sns_initial_history_ready");
          }
          setTimeout(function() {
            var node = $topic.get(0);
            if (node) {
              node.scrollTop = node.scrollHeight;
            }
          }, 220);
        }
        function fallbackToHtmlPages() {
          if (failed) {
            return;
          }
          failed = true;
          $("#sns-pages-loading").remove();
          syncPagedHistoryFallback();
        }
        function fetchNextBatch() {
          if (failed) {
            return;
          }
          if (skip > MAX_SKIP) {
            fallbackToHtmlPages();
            return;
          }
          fetchHistoryApiBatch(topicIdValue, skip).then(function(posts) {
            if (failed) {
              return;
            }
            if (!posts.length) {
              if (expectedRaw > fetchedRaw) {
                fallbackToHtmlPages();
                return;
              }
              finalizeApiHistory();
              return;
            }
            var stats = appendApiHistoryBatch($topic, posts);
            fetchedRaw += stats.raw;
            fetchedConfigs += stats.configs;
            addedReal += stats.added;
            updateStatus();
            if (expectedRaw > 0 && fetchedRaw >= expectedRaw) {
              finalizeApiHistory();
              return;
            }
            if (expectedRaw <= 0 && posts.length < 100) {
              finalizeApiHistory();
              return;
            }
            if (posts.length < 100 && expectedRaw > fetchedRaw) {
              fallbackToHtmlPages();
              return;
            }
            skip += 100;
            setTimeout(fetchNextBatch, 80);
          }).catch(fallbackToHtmlPages);
        }
        fetchTopicPostCount(topicIdValue).then(function(count) {
          expectedRaw = count;
          updateStatus();
          fetchNextBatch();
        });
      }
      function install() {
        if (!$composer().length || !$wrap().length || !$input().length) {
          return false;
        }
        $composer().children(".sns-chat-resize-handle," + ".sns-composer-resize-handle," + ".sns-composer-resize-handle-v37").remove();
        $composer().css("align-items", "end");
        bindManualOverflow();
        bindResize();
        manualHeight = MIN_H;
        setRealHeight(MIN_H);
        updateOverflow(MIN_H);
        return true;
      }
      var attempts = 0;
      var timer = setInterval(function() {
        attempts++;
        if (install() || attempts > 80) {
          clearInterval(timer);
        }
      }, 100);
    })(jQuery);
    (function() {
      if (document.getElementById("sns-v39-manual-textarea-resize-style")) {
        return;
      }
      var style = document.createElement("style");
      style.id = "sns-v39-manual-textarea-resize-style";
      style.textContent = [ "body.sns-chat-page #sns-composer-ui{", "align-items:end!important;", "padding-bottom:18px!important;", "}", "body.sns-chat-page #sns-composer-ui>.sns-ui-input-wrap{", "align-self:end!important;", "box-sizing:border-box!important;", "width:100%!important;", "}", "body.sns-chat-page #sns-ui-input{", "display:block!important;", "box-sizing:border-box!important;", "width:100%!important;", "resize:none!important;", "overflow-x:hidden!important;", "}", "body.sns-chat-page #sns-composer-ui>.sns-ui-plus,", "body.sns-chat-page #sns-composer-ui>.sns-ui-format{", "align-self:center!important;", "}", "body.sns-chat-page #sns-composer-ui>.sns-ui-send{", "align-self:end!important;", "}", "body.sns-chat-page #sns-composer-ui>.sns-composer-resize-handle-v38{", "position:absolute!important;", "left:50%!important;", "bottom:0!important;", "z-index:2000!important;", "display:block!important;", "width:180px!important;", "height:18px!important;", "margin:0!important;", "padding:0!important;", "transform:translateX(-50%)!important;", "border:0!important;", "outline:0!important;", "background:transparent!important;", "opacity:0!important;", "cursor:ns-resize!important;", "pointer-events:auto!important;", "}", "body.sns-chat-page #sns-composer-ui>.sns-composer-resize-handle-v38>span{", "display:none!important;", "}", "body.sns-composer-v38-dragging,", "body.sns-composer-v38-dragging *{", "cursor:ns-resize!important;", "user-select:none!important;", "}", "@media (max-width:650px){", "body.sns-chat-page #sns-composer-ui>.sns-composer-resize-handle-v38{", "width:150px!important;height:20px!important;", "}", "}" ].join("");
      document.head.appendChild(style);
    })();
    (function($) {
      "use strict";
      function cleanText(value) {
        return String(value || "").replace(/\s+/g, " ").trim();
      }
      function topicId() {
        var match = String(location.search || "").match(/(?:\?|&)id=(\d+)/);
        return match ? match[1] : "";
      }
      function pageNumberFromUrl(href) {
        var match = String(href || "").match(/(?:\?|&)p=(\d+)/);
        return match ? parseInt(match[1], 10) : 1;
      }
      function maxTopicPage() {
        var id = topicId();
        if (!id) {
          return 1;
        }
        var maxPage = 1;
        $("a[href]").each(function() {
          var href = this.href || $(this).attr("href") || "";
          if (!/viewtopic\.php/i.test(href)) {
            return;
          }
          var idMatch = href.match(/(?:\?|&)id=(\d+)/);
          if (!idMatch || idMatch[1] !== id) {
            return;
          }
          maxPage = Math.max(maxPage, pageNumberFromUrl(href));
        });
        return maxPage;
      }
      function pageUrl(pageNumber) {
        var url = new URL(location.href);
        url.hash = "";
        if (pageNumber <= 1) {
          url.searchParams.delete("p");
        } else {
          url.searchParams.set("p", String(pageNumber));
        }
        return url.toString();
      }
      function postKey($post) {
        var id = String($post.attr("id") || "").trim();
        if (id) {
          return id;
        }
        var href = $post.find('a[href*="#p"]').first().attr("href") || "";
        var match = href.match(/#(p\d+)/);
        return match ? match[1] : "";
      }
      function isConfigPost($post) {
        var content = String($post.find(".post-content").first().text() || "");
        return /SNSCFG:[A-Za-z0-9+\/=]+/.test(content);
      }
      function existingPostKeys($topic) {
        var seen = {};
        $topic.find(".post").each(function() {
          var key = postKey($(this));
          if (key) {
            seen[key] = true;
          }
        });
        return seen;
      }
      function parseHistoryDocument(html) {
        return (new DOMParser).parseFromString(String(html || ""), "text/html");
      }
      function parsedPosts(html) {
        var doc = parseHistoryDocument(html);
        var topic = doc.querySelector("#pun-viewtopic .topic");
        if (!topic) return $();
        topic.querySelectorAll("img").forEach(function(image) {
          image.setAttribute("loading", "lazy");
          image.setAttribute("decoding", "async");
        });
        topic.querySelectorAll("audio,video").forEach(function(media) {
          media.setAttribute("preload", "none");
          media.removeAttribute("autoplay");
        });
        return $(Array.prototype.slice.call(topic.querySelectorAll(".post")));
      }
      function maxTopicPageFromHtml(html) {
        var id = topicId();
        if (!id) {
          return 1;
        }
        var doc = parseHistoryDocument(html);
        var maxPage = 1;
        $(Array.prototype.slice.call(doc.querySelectorAll("a[href]"))).each(function() {
          var href = $(this).attr("href") || "";
          if (!/viewtopic\.php/i.test(href)) {
            return;
          }
          var idMatch = href.match(/(?:\?|&)id=(\d+)/);
          if (!idMatch || idMatch[1] !== id) {
            return;
          }
          maxPage = Math.max(maxPage, pageNumberFromUrl(href));
        });
        return maxPage;
      }
      function appendPostsFromHtml($topic, html) {
        if (!$topic || !$topic.length) {
          return 0;
        }
        var seen = existingPostKeys($topic);
        var fragment = document.createDocumentFragment();
        var added = 0;
        parsedPosts(html).each(function() {
          var $post = $(this);
          var key = postKey($post);
          if (isConfigPost($post)) {
            return;
          }
          if (key && seen[key]) {
            return;
          }
          if (key) {
            seen[key] = true;
          }
          fragment.appendChild(this);
          added++;
        });
        if (added && $topic.get(0)) {
          $topic.get(0).appendChild(fragment);
          window.__SNS_LAST_HISTORY_ADD_AT__ = Date.now();
          window.__SNS_LAST_HISTORY_ADD_COUNT__ = added;
        }
        return added;
      }
      var latestRefreshRunning = false;
      var latestRefreshQueued = false;
      function publishedNoticeData() {
        var result = null;
        $("a[href]").each(function() {
          var $link = $(this);
          var linkText = cleanText($link.text()).toLowerCase();
          var href = $link.attr("href") || "";
          if (linkText.indexOf("\u043f\u043e\u0441\u043b\u0435\u0434\u043d") === -1 || !/viewtopic\.php/i.test(href)) {
            return;
          }
          var $candidate = $link;
          for (var level = 0; level < 6; level++) {
            var $parent = $candidate.parent();
            if (!$parent.length) {
              break;
            }
            var parentText = cleanText($parent.text());
            if (parentText.length > 400) {
              break;
            }
            if (parentText.toLowerCase().indexOf("\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435 \u043e\u043f\u0443\u0431\u043b\u0438\u043a\u043e\u0432\u0430\u043d\u043e") !== -1) {
              $candidate = $parent;
            } else {
              break;
            }
          }
          result = {
            href: new URL(href, location.href).toString(),
            $notice: $candidate
          };
          return false;
        });
        return result;
      }
      var exactPostFetchRunning = false;
      function fetchPublishedNoticePost() {
        var notice = publishedNoticeData();
        if (!notice) {
          return false;
        }
        if (notice.$notice && notice.$notice.length) {
          notice.$notice.css("display", "none");
        }
        if (exactPostFetchRunning) {
          return true;
        }
        exactPostFetchRunning = true;
        var requestUrl = new URL(notice.href);
        requestUrl.searchParams.set("_sns_exact", String(Date.now()));
        SNSRequest({
          url: requestUrl.toString(),
          type: "GET",
          dataType: "html",
          cache: false,
          timeout: 5500
        }).done(function(html) {
          var $topic = $("#sns-chat-shell>.topic").first();
          if (!$topic.length) {
            return;
          }
          var added = appendPostsFromHtml($topic, html);
          if (added) {
            $("#sns-empty").remove();
            $(document).trigger("sns_pages_loaded");
            setTimeout(function() {
              var node = $topic.get(0);
              if (node) {
                node.scrollTop = node.scrollHeight;
              }
            }, 120);
          }
        }).always(function() {
          exactPostFetchRunning = false;
        });
        return true;
      }
      window.SNSFetchPublishedNoticePost = fetchPublishedNoticePost;
      function installPublishedNoticeObserver() {
        if (window.__SNS_PUBLISHED_NOTICE_OBSERVER__) {
          return;
        }
        var observer = new MutationObserver(function() {
          if (publishedNoticeData()) {
            fetchPublishedNoticePost();
          }
        });
        observer.observe(document.body, {
          childList: true,
          subtree: true
        });
        window.__SNS_PUBLISHED_NOTICE_OBSERVER__ = observer;
      }
      function apiTopicId() {
        return topicId();
      }
      function apiPostExists(postId) {
        postId = String(postId || "");
        if (!postId) {
          return true;
        }
        return !!apiPostReserved[postId] || $("#sns-chat-shell>.topic").find("#p" + postId).length > 0;
      }
      function apiPostTime(posted) {
        var timestamp = parseInt(posted, 10);
        if (!timestamp) {
          return "";
        }
        var date = new Date(timestamp * 1e3);
        var now = new Date;
        function pad(value) {
          return String(value).padStart(2, "0");
        }
        var time = pad(date.getHours()) + ":" + pad(date.getMinutes()) + ":" + pad(date.getSeconds());
        var sameDay = date.getFullYear() === now.getFullYear() && date.getMonth() === now.getMonth() && date.getDate() === now.getDate();
        if (sameDay) {
          return "\u0421\u0435\u0433\u043e\u0434\u043d\u044f " + time;
        }
        return pad(date.getDate()) + "." + pad(date.getMonth() + 1) + "." + date.getFullYear() + " " + time;
      }
      function apiMessageIsConfig(message) {
        var temp = document.createElement("div");
        temp.innerHTML = String(message || "");
        var raw = String(temp.textContent || temp.innerText || "");
        return /SNSCFG:[A-Za-z0-9+\/=]+/.test(raw);
      }
      function createApiPostNode(post) {
        var id = String(post.id || "");
        var userId = String(post.user_id || "");
        var username = String(post.username || "");
        var message = String(post.message || "");
        var time = apiPostTime(post.posted);
        var $post = $('<div class="post"></div>').attr("id", "p" + id).attr("data-user-id", userId).attr("data-sns-api-post", "1");
        var $container = $('<div class="container"></div>').appendTo($post);
        var $author = $('<div class="post-author"><ul></ul></div>').appendTo($container);
        var $authorLi = $('<li class="pa-author"></li>').appendTo($author.find("ul"));
        $("<a></a>").attr("href", "/profile.php?id=" + encodeURIComponent(userId)).text(username).appendTo($authorLi);
        var $body = $('<div class="post-body"></div>').appendTo($container);
        $("<h3></h3>").text(time).appendTo($body);
        var $box = $('<div class="post-box"></div>').appendTo($body);
        $('<div class="post-content"></div>').html(message).appendTo($box);
        var $links = $('<div class="post-links"></div>').appendTo($container);
        $("<a></a>").attr("href", "/edit.php?id=" + encodeURIComponent(id)).text("\u0440\u0435\u0434\u0430\u043a\u0442\u0438\u0440\u043e\u0432\u0430\u0442\u044c").appendTo($links);
        $("<a></a>").attr("href", "/delete.php?id=" + encodeURIComponent(id)).text("\u0443\u0434\u0430\u043b\u0438\u0442\u044c").appendTo($links);
        setTimeout(function() {
          if (typeof window.SNSResolveUserVisual !== "function") {
            return;
          }
          window.SNSResolveUserVisual(userId, username, function(data) {
            if (!data || !data.avatar) {
              return;
            }
            var $livePost = $("#sns-chat-shell>.topic").find("#p" + id).first();
            if (!$livePost.length) {
              return;
            }
            var $author = $livePost.find(".post-author ul").first();
            if (!$author.length) {
              return;
            }
            var $avatar = $author.find(".pa-avatar").first();
            if (!$avatar.length) {
              $avatar = $('<li class="pa-avatar"></li>').appendTo($author);
            }
            $avatar.empty().append($('<img alt="">').attr("src", data.avatar));
            if (data.name) {
              $livePost.find(".pa-author a").first().text(data.name);
            }
            $(document).trigger("sns_user_visual_loaded");
          });
        }, 0);
        return $post;
      }
      var apiPostRefreshRunning = false;
      var apiPostRefreshQueued = false;
      var apiPostReserved = {};
      function refreshLatestPostsFromApi() {
        if (apiPostRefreshRunning) {
          apiPostRefreshQueued = true;
          return;
        }
        var id = apiTopicId();
        var $topic = $("#sns-chat-shell>.topic").first();
        if (!id || !$topic.length) {
          return;
        }
        apiPostRefreshRunning = true;
        var apiUrl = new URL("api.php", location.href);
        apiUrl.searchParams.set("method", "post.get");
        apiUrl.searchParams.set("topic_id", id);
        apiUrl.searchParams.set("sort_by", "id");
        apiUrl.searchParams.set("sort_dir", "desc");
        apiUrl.searchParams.set("limit", "30");
        apiUrl.searchParams.set("fields", "id,username,user_id,message,posted,topic_id,avatar");
        apiUrl.searchParams.set("_sns", String(Date.now()));
        SNSRequest({
          url: apiUrl.toString(),
          type: "GET",
          dataType: "json",
          cache: false,
          timeout: 5e3
        }).done(function(json) {
          var posts = json && Array.isArray(json.response) ? json.response.slice() : [];
          if (!posts.length) {
            return;
          }
          posts.sort(function(a, b) {
            return parseInt(a.id, 10) - parseInt(b.id, 10);
          });
          var added = 0;
          posts.forEach(function(post) {
            if (!post || !post.id || apiPostExists(post.id) || apiMessageIsConfig(post.message)) {
              return;
            }
            var reservedId = String(post.id);
            apiPostReserved[reservedId] = true;
            var $newPost = createApiPostNode(post);
            $topic.append($newPost);
            delete apiPostReserved[reservedId];
            added++;
          });
          if (added) {
            window.__SNS_LAST_HISTORY_ADD_AT__ = Date.now();
            window.__SNS_LAST_HISTORY_ADD_COUNT__ = added;
            $("#sns-empty").remove();
            setTimeout(function() {
              $(document).trigger("sns_pages_loaded");
              var node = $topic.get(0);
              if (node) {
                node.scrollTop = node.scrollHeight;
              }
            }, 120);
          }
        }).always(function() {
          apiPostRefreshRunning = false;
          if (apiPostRefreshQueued) {
            apiPostRefreshQueued = false;
            setTimeout(refreshLatestPostsFromApi, 100);
          }
        });
      }
      window.SNSRefreshLatestPostsAPI = refreshLatestPostsFromApi;
      function hidePublishedRusffNotice() {
        var publishedPhrase = "\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435 \u043e\u043f\u0443\u0431\u043b\u0438\u043a\u043e\u0432\u0430\u043d\u043e";
        var lastPagePhrase = "\u043f\u043e\u0441\u043b\u0435\u0434\u043d\u0435\u0439 \u0441\u0442\u0440\u0430\u043d\u0438\u0446";
        $(".jGrowl-notification," + ".jGrowl-notice," + "#jGrowl > div," + "#jGrowl .jGrowl-notification," + ".punbb-notification," + ".notification," + ".notice," + ".message-popup," + ".popup-message," + '[role="alert"],' + '[role="status"]').each(function() {
          var $node = $(this);
          var value = cleanText($node.text()).toLowerCase();
          if (value.indexOf(publishedPhrase) !== -1 && value.indexOf(lastPagePhrase) !== -1) {
            $(document).trigger("sns_forum_publish_confirmed");
            $node.stop(true, true).remove();
          }
        });
        $("a").each(function() {
          var $link = $(this);
          var linkText = cleanText($link.text()).toLowerCase();
          if (linkText.indexOf(lastPagePhrase) === -1) {
            return;
          }
          var $candidate = $link;
          for (var level = 0; level < 8; level++) {
            var $parent = $candidate.parent();
            if (!$parent.length) {
              break;
            }
            var textValue = cleanText($parent.text()).toLowerCase();
            if (textValue.indexOf(publishedPhrase) !== -1) {
              var rect = $parent.get(0) ? $parent.get(0).getBoundingClientRect() : null;
              if (!rect || rect.width <= 700 && rect.height <= 250) {
                $(document).trigger("sns_forum_publish_confirmed");
                $parent.stop(true, true).remove();
                return false;
              }
            }
            $candidate = $parent;
          }
        });
      }
      function installPublishedNoticeSuppressor() {
        if (window.__SNS_NOTICE_SUPPRESSOR_V59__) {
          return;
        }
        window.__SNS_NOTICE_SUPPRESSOR_V59__ = {
          eventDriven: true
        };
        hidePublishedRusffNotice();
      }
      function refreshLatestHistory() {
        if (latestRefreshRunning) {
          latestRefreshQueued = true;
          return;
        }
        var $topic = $("#sns-chat-shell>.topic").first();
        if (!$topic.length) {
          return;
        }
        latestRefreshRunning = true;
        var rootUrl = pageUrl(1);
        var rootRequestUrl = new URL(rootUrl);
        rootRequestUrl.searchParams.set("_sns_refresh", String(Date.now()));
        SNSRequest({
          url: rootRequestUrl.toString(),
          type: "GET",
          dataType: "html",
          cache: false,
          timeout: 5e3
        }).done(function(rootHtml) {
          var freshMaxPage = Math.max(1, maxTopicPage(), maxTopicPageFromHtml(rootHtml));
          if (freshMaxPage <= 1) {
            appendPostsFromHtml($topic, rootHtml);
            return;
          }
          var lastUrl = pageUrl(freshMaxPage);
          var lastRequestUrl = new URL(lastUrl);
          lastRequestUrl.searchParams.set("_sns_refresh", String(Date.now()));
          return SNSRequest({
            url: lastRequestUrl.toString(),
            type: "GET",
            dataType: "html",
            cache: false,
            timeout: 5e3
          }).done(function(lastHtml) {
            var added = appendPostsFromHtml($topic, lastHtml);
            if (!added) {
              appendPostsFromHtml($topic, rootHtml);
            }
          });
        }).always(function() {
          setTimeout(function() {
            $("#sns-empty").remove();
            $(document).trigger("sns_pages_loaded");
            if (!window.__SNS_INITIAL_HISTORY_READY__) {
              window.__SNS_INITIAL_HISTORY_READY__ = true;
              $(document).trigger("sns_initial_history_ready");
            }
            var topicNode = $topic.get(0);
            if (topicNode) {
              topicNode.scrollTop = topicNode.scrollHeight;
            }
            latestRefreshRunning = false;
            if (latestRefreshQueued) {
              latestRefreshQueued = false;
              setTimeout(refreshLatestHistory, 80);
            }
          }, 180);
        });
      }
      window.SNSRefreshLatestHistory = refreshLatestHistory;
      function numericPostId($post) {
        var raw = String($post.attr("id") || "");
        var match = raw.match(/^p(\d+)$/);
        return match ? parseInt(match[1], 10) : 0;
      }
      function sortTopicPostsChronologically($topic) {
        if (!$topic || !$topic.length) return;
        var root = $topic.get(0);
        var posts = $topic.children(".post").get().sort(function(a, b) {
          return (numericPostId($(a)) || Infinity) - (numericPostId($(b)) || Infinity);
        });
        var current = $topic.children(".post").get();
        var same = current.length === posts.length && current.every(function(post, index) {
          return post === posts[index];
        });
        if (!same) {
          var cursor = current[0] || null;
          posts.forEach(function(post) {
            if (post === cursor) {
              cursor = cursor.nextElementSibling;
              while (cursor && !cursor.classList.contains("post")) cursor = cursor.nextElementSibling;
            } else root.insertBefore(post, cursor);
          });
        }
        var pending = root.querySelector(":scope > #sns-pending-stack");
        if (pending && pending !== root.lastElementChild) root.appendChild(pending);
      }
      function syncPagedHistoryFallback() {
        var $topic = $("#sns-chat-shell>.topic").first();
        if (!$topic.length) {
          return;
        }
        var $loading = $("#sns-pages-loading");
        if (!$loading.length) {
          $loading = $('<div id="sns-pages-loading">' + "\u0437\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0438\u0441\u0442\u043e\u0440\u0438\u044e\u2026" + "</div>");
          $topic.append($loading);
        }
        var pageResults = {};
        window.__SNS_HISTORY_PAGE_RESULTS__ = pageResults;
        var failedPages = [];
        var maxPage = 1;
        var finished = false;
        var activeRequests = [];
        var pageRequests = {};
        var MAX_CONCURRENT = 2;
        var RETRIES_PER_PAGE = 2;
        function updateLoading(done, total) {
          if (!$loading.length || !$loading.closest("html").length) {
            $loading = $("#sns-pages-loading");
          }
          if (!$loading.length) {
            return;
          }
          if (total > 1) {
            $loading.text("\u0437\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0438\u0441\u0442\u043e\u0440\u0438\u044e " + done + "/" + total);
          } else {
            $loading.text("\u0437\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0438\u0441\u0442\u043e\u0440\u0438\u044e\u2026");
          }
        }
        function historyPageUrl(pageNumber, attempt) {
          var url = new URL(pageUrl(pageNumber));
          url.searchParams.set("_sns_history", String(Date.now()) + "_" + pageNumber + "_" + attempt);
          return url.toString();
        }
        function fetchPage(pageNumber, attempt) {
          attempt = attempt || 1;
          if (pageResults[pageNumber]) return $.Deferred().resolve(pageResults[pageNumber]).promise();
          if (!attempt || attempt === 1) {
            if (pageRequests[pageNumber]) return pageRequests[pageNumber];
          }
          var request = SNSRequest({
            url: historyPageUrl(pageNumber, attempt),
            type: "GET",
            dataType: "html",
            cache: false,
            timeout: 8e3
          });
          activeRequests.push(request);
          request.always(function() {
            var index = activeRequests.indexOf(request);
            if (index !== -1) activeRequests.splice(index, 1);
          });
          var result = request.then(function(html) {
            var count = parsedPosts(html).length;
            window.__SNS_LAST_PHYSICAL_FETCH__ = {
              page: pageNumber,
              posts: count,
              ok: !!count,
              at: Date.now()
            };
            if (!count) {
              return $.Deferred().reject("empty-history-page").promise();
            }
            pageResults[pageNumber] = html;
            return html;
          }).then(null, function() {
            if (attempt < RETRIES_PER_PAGE) {
              return new Promise(function(resolve) {
                setTimeout(resolve, 450 * attempt);
              }).then(function() {
                return fetchPage(pageNumber, attempt + 1);
              });
            }
            return $.Deferred().reject("history-page-failed").promise();
          });
          if (attempt === 1) {
            pageRequests[pageNumber] = result;
            result.always(function() {
              if (pageRequests[pageNumber] === result) delete pageRequests[pageNumber];
            });
          }
          return result;
        }
        var INITIAL_VISIBLE_HISTORY_PAGES = 1;
        var lazyOldestLoadedPage = null;
        var lazyHistoryLoading = false;
        var lazyHistoryInstalled = false;
        function removeDisplayedNonConfigPosts() {
          $topic.children(".post").each(function() {
            var $post = $(this);
            if (isConfigPost($post)) {
              return;
            }
            $post.remove();
          });
          $topic.children(".sns-story-date-separator, .sns-date-separator").remove();
        }
        function historyLoaderNode() {
          var $loader = $("#sns-history-loader-v205");
          if (!$loader.length) {
            $loader = $("<button " + 'type="button" ' + 'id="sns-history-loader-v205" ' + 'aria-label="\u0417\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u043f\u0440\u0435\u0434\u044b\u0434\u0443\u0449\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f">' + "<span>\u043f\u0440\u0435\u0434\u044b\u0434\u0443\u0449\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f</span>" + "</button>");
            $topic.prepend($loader);
          }
          return $loader;
        }
        function updateHistoryLoader() {
          var $loader = $("#sns-history-loader-v205");
          if (lazyOldestLoadedPage === null || lazyOldestLoadedPage <= 1) {
            $loader.remove();
            return;
          }
          $loader = historyLoaderNode();
          $loader.prop("disabled", lazyHistoryLoading).toggleClass("is-loading", lazyHistoryLoading).find("span").text(lazyHistoryLoading ? "\u0437\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u043f\u0440\u0435\u0434\u044b\u0434\u0443\u0449\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f\u2026" : "\u043f\u0440\u0435\u0434\u044b\u0434\u0443\u0449\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f");
        }
        function firstVisibleHistoryPost() {
          var topicNode = $topic.get(0);
          if (!topicNode) {
            return null;
          }
          var topicRect = topicNode.getBoundingClientRect();
          var found = null;
          $topic.children(".post").each(function() {
            var rect = this.getBoundingClientRect();
            if (rect.bottom > topicRect.top + 2) {
              found = this;
              return false;
            }
          });
          return found;
        }
        function restoreHistoryAnchor(anchorNode, anchorTop) {
          if (!anchorNode || !document.documentElement.contains(anchorNode)) {
            return;
          }
          var topicNode = $topic.get(0);
          if (!topicNode) {
            return;
          }
          var adjust = function() {
            if (!document.documentElement.contains(anchorNode)) {
              return;
            }
            var newTop = anchorNode.getBoundingClientRect().top;
            var delta = newTop - anchorTop;
            if (isFinite(delta) && Math.abs(delta) > .5) {
              topicNode.scrollTop += delta;
            }
          };
          if (typeof window.requestAnimationFrame === "function") {
            window.requestAnimationFrame(function() {
              adjust();
              window.requestAnimationFrame(adjust);
            });
          } else {
            setTimeout(adjust, 30);
            setTimeout(adjust, 110);
          }
        }
        function appendOlderHistoryPage(pageNumber, html) {
          var anchorNode = firstVisibleHistoryPost();
          var anchorTop = anchorNode ? anchorNode.getBoundingClientRect().top : 0;
          window.__SNS_LAZY_HISTORY_PREPENDING__ = true;
          pageResults[pageNumber] = html;
          appendPostsFromHtml($topic, html);
          lazyOldestLoadedPage = Math.min(lazyOldestLoadedPage === null ? pageNumber : lazyOldestLoadedPage, pageNumber);
          $topic.children('.sns-search-temp[data-sns-source-page="' + pageNumber + '"]').removeClass("sns-search-temp").removeAttr("data-sns-source-page");
          sortTopicPostsChronologically($topic);
          updateHistoryLoader();
          $(document).trigger("sns_pages_loaded");
          restoreHistoryAnchor(anchorNode, anchorTop);
          window.__SNS_LAZY_HISTORY_OLDEST_PAGE__ = lazyOldestLoadedPage;
          setTimeout(function() {
            window.__SNS_LAZY_HISTORY_PREPENDING__ = false;
          }, 240);
        }
        function loadPreviousHistoryPage() {
          if (lazyHistoryLoading || lazyOldestLoadedPage === null || lazyOldestLoadedPage <= 1) {
            return;
          }
          var targetPage = lazyOldestLoadedPage - 1;
          lazyHistoryLoading = true;
          updateHistoryLoader();
          var finish = function(html) {
            appendOlderHistoryPage(targetPage, html);
            lazyHistoryLoading = false;
            updateHistoryLoader();
          };
          var fail = function() {
            lazyHistoryLoading = false;
            updateHistoryLoader();
          };
          if (pageResults[targetPage]) {
            finish(pageResults[targetPage]);
            return;
          }
          fetchPage(targetPage, 1).then(finish, fail);
        }
        function installLazyHistoryControls() {
          if (lazyHistoryInstalled) {
            return;
          }
          lazyHistoryInstalled = true;
          var lastScrollTop = $topic.get(0) ? $topic.get(0).scrollTop : 0;
          var userUpIntentUntil = 0;
          var lastLazyLoadAt = 0;
          var LAZY_TOP_THRESHOLD = 42;
          var LAZY_LOAD_COOLDOWN = 650;
          function armUpIntent(duration) {
            userUpIntentUntil = Date.now() + (duration || 900);
          }
          $topic.off("wheel.snsLazyHistoryIntentV240").on("wheel.snsLazyHistoryIntentV240", function(event) {
            var original = event.originalEvent || event;
            if (original && Number(original.deltaY) < 0) {
              armUpIntent(950);
            }
          });
          $topic.off("keydown.snsLazyHistoryIntentV240").on("keydown.snsLazyHistoryIntentV240", function(event) {
            if (event.key === "ArrowUp" || event.key === "PageUp" || event.key === "Home") {
              armUpIntent(1200);
            }
          });
          $topic.off("touchstart.snsLazyHistoryIntentV240 " + "touchmove.snsLazyHistoryIntentV240").on("touchstart.snsLazyHistoryIntentV240 " + "touchmove.snsLazyHistoryIntentV240", function() {
            armUpIntent(900);
          });
          $topic.off("pointerdown.snsLazyHistoryIntentV240").on("pointerdown.snsLazyHistoryIntentV240", function() {
            armUpIntent(1200);
          });
          $topic.off("scroll.snsLazyHistoryV205").on("scroll.snsLazyHistoryV205", function() {
            var currentTop = Number(this.scrollTop) || 0;
            var movedUp = currentTop < lastScrollTop - 1;
            lastScrollTop = currentTop;
            if (Date.now() > userUpIntentUntil) {
              return;
            }
            if (!movedUp || currentTop > LAZY_TOP_THRESHOLD || lazyHistoryLoading || window.__SNS_LAZY_HISTORY_PREPENDING__ === true) {
              return;
            }
            if (Date.now() - lastLazyLoadAt < LAZY_LOAD_COOLDOWN) {
              return;
            }
            lastLazyLoadAt = Date.now();
            userUpIntentUntil = 0;
            loadPreviousHistoryPage();
          });
          $topic.off("click.snsLazyHistoryV205", "#sns-history-loader-v205").on("click.snsLazyHistoryV205", "#sns-history-loader-v205", function(event) {
            event.preventDefault();
            lastLazyLoadAt = Date.now();
            loadPreviousHistoryPage();
          });
        }
        function searchablePayloadStrings(value, output) {
          output = output || [];
          if (value === null || typeof value === "undefined") {
            return output;
          }
          if (typeof value === "string") {
            var stringValue = String(value).replace(/\s+/g, " ").trim();
            if (stringValue && !/^https?:\/\//i.test(stringValue) && !/^[A-Za-z0-9+\/=]{42,}$/.test(stringValue)) {
              output.push(stringValue);
            }
            return output;
          }
          if (Array.isArray(value)) {
            value.forEach(function(item) {
              searchablePayloadStrings(item, output);
            });
            return output;
          }
          if (typeof value === "object") {
            Object.keys(value).forEach(function(key) {
              if (/^(url|src|secure_url|playback_url|cover|image|imageUrl|id|postId|userId)$/i.test(key)) {
                return;
              }
              searchablePayloadStrings(value[key], output);
            });
          }
          return output;
        }
        function decodeSearchMarkerPayload(encoded) {
          try {
            var binary = atob(String(encoded || ""));
            var bytes = new Uint8Array(binary.length);
            for (var i = 0; i < binary.length; i++) {
              bytes[i] = binary.charCodeAt(i);
            }
            var jsonText;
            if (typeof TextDecoder !== "undefined") {
              jsonText = new TextDecoder("utf-8").decode(bytes);
            } else {
              var escaped = "";
              for (var j = 0; j < bytes.length; j++) {
                escaped += "%" + ("0" + bytes[j].toString(16)).slice(-2);
              }
              jsonText = decodeURIComponent(escaped);
            }
            var parsed = JSON.parse(jsonText);
            return searchablePayloadStrings(parsed, []).join(" ");
          } catch (error) {
            return "";
          }
        }
        var SEARCH_UI_SELECTOR = ".sns-controls,.sns-reaction-chips,.sns-reaction-control,.sns-reaction-picker-custom,.sns-menu,.sns-meta,.sns-mini-avatar,.sns-inline-editor,.sns-audio-controls,.sns-audio-error,.sns-video-error-note";
        function searchableTextFromPost($post, live) {
          var content = $post.find(".post-content").first().get(0);
          var raw = "";
          if (content && live) {
            // Skip UI subtrees without cloning an entire rendered message.
            var walker = content.ownerDocument.createTreeWalker(content, 5, {
              acceptNode: function(node) {
                if (node.nodeType === 1) {
                  return $(node).is(SEARCH_UI_SELECTOR) ? 2 : 3;
                }
                return 1;
              }
            }, false);
            var part;
            var textParts = [];
            var previousAudioField = null;
            while (part = walker.nextNode()) {
              var audioField = $(part.parentNode).closest(".sns-audio-title,.sns-audio-artist").get(0) || null;
              // Audio fields need boundaries; normal inline text stays contiguous.
              if (audioField !== previousAudioField && (audioField || previousAudioField)) textParts.push(" ");
              textParts.push(part.nodeValue || "");
              previousAudioField = audioField;
            }
            raw = textParts.join("");
          } else if (content) {
            raw = String(content.textContent || "");
          }
          var decoded = [];
          raw = raw.replace(/\bSNS[A-Z0-9_]*:([A-Za-z0-9+\/=]+)/g, function(whole, encoded) {
            var payloadText = decodeSearchMarkerPayload(encoded);
            if (payloadText) {
              decoded.push(payloadText);
            }
            return " ";
          });
          raw = raw.replace(/\s+/g, " ").trim();
          return (raw + (decoded.length ? " " + decoded.join(" ") : "")).replace(/\s+/g, " ").trim();
        }
        function searchAuthorFromPost($post) {
          var author = String($post.find(".sns-author-name").first().text() || "");
          try {
            author = author || getPostAuthor($post) || "";
          } catch (error) {
            author = author || "";
          }
          if (!author) {
            author = String($post.find(".pa-author a, .pa-author, .post-author a, .post-author strong, .post-author").first().text() || "");
          }
          return author.replace(/\s+/g, " ").trim();
        }
        var searchPageCache = Object.create(null);
        var searchPostCache = typeof WeakMap === "function" ? new WeakMap : null;
        var searchPostCacheKey = "__snsHistorySearchCacheV242";
        var searchObserver = null;
        function postSearchCache(post) {
          return searchPostCache ? searchPostCache.get(post) : post[searchPostCacheKey];
        }
        function savePostSearchCache(post, cache) {
          if (searchPostCache) searchPostCache.set(post, cache); else post[searchPostCacheKey] = cache;
        }
        function invalidateSearchMutations(records) {
          var changed = false;
          records.forEach(function(record) {
            var target = record.target.nodeType === 1 ? record.target : record.target.parentNode;
            if (!target) return;
            // Rendered author names live inside the otherwise excluded .sns-meta.
            var authorChanged = !!$(target).closest(".sns-author-name").length;
            if (!authorChanged && record.type === "childList") {
              [ record.addedNodes, record.removedNodes ].forEach(function(nodes) {
                for (var i = 0; i < nodes.length; i++) {
                  if (nodes[i].nodeType === 1 && ($(nodes[i]).is(".sns-author-name") || nodes[i].querySelector(".sns-author-name"))) authorChanged = true;
                }
              });
            }
            if (!authorChanged && $(target).closest(SEARCH_UI_SELECTOR).length) return;
            var post = $(target).closest(".post").get(0);
            if (!post) {
              if (target === $topic.get(0) && record.type === "childList") {
                [ record.addedNodes, record.removedNodes ].forEach(function(nodes) {
                  for (var i = 0; i < nodes.length; i++) {
                    if (nodes[i].nodeType === 1 && $(nodes[i]).is(".post")) changed = true;
                  }
                });
              }
              return;
            }
            changed = true;
            var cache = postSearchCache(post);
            if (cache) cache.dirty = true;
          });
          return changed;
        }
        function ensureSearchObserver() {
          if (searchObserver || typeof window.MutationObserver !== "function") return;
          searchObserver = new window.MutationObserver(function(records) {
            if (invalidateSearchMutations(records)) $(document).trigger("sns_search_index_changed");
          });
          searchObserver.observe($topic.get(0), {
            childList: true,
            characterData: true,
            subtree: true,
            attributes: true,
            attributeFilter: [ "id", "class" ]
          });
        }
        function cachedLiveSearchItem(post) {
          var $post = $(post);
          var cache = postSearchCache(post);
          // Index committed content while an inline editor is open.
          if ($post.hasClass("sns-inline-edit-open")) return cache ? cache.item : null;
          var signature = "";
          if (!searchObserver) {
            // Legacy fallback remains fresh even when forum events are omitted.
            signature = String($post.find(".post-content").first().text() || "") + "\n" + searchAuthorFromPost($post) + "\n" + postKey($post);
          }
          if (cache && !cache.dirty && (searchObserver || cache.signature === signature)) return cache.item;
          var key = postKey($post);
          var text = key && !isConfigPost($post) ? searchableTextFromPost($post, true) : "";
          var item = key ? {
            postId: key,
            author: searchAuthorFromPost($post),
            text: text,
            lowerText: text.toLocaleLowerCase()
          } : null;
          savePostSearchCache(post, { item: item, dirty: false, signature: signature });
          return item;
        }
        function cachedPageSearchItems(pageNumber) {
          var html = pageResults[pageNumber];
          var cache = searchPageCache[pageNumber];
          if (cache && cache.html === html) return cache.items;
          var items = [];
          parsedPosts(html).each(function() {
            var $post = $(this);
            var key = postKey($post);
            if (!key || isConfigPost($post)) return;
            var text = searchableTextFromPost($post);
            items.push({
              postId: key,
              page: Number(pageNumber) || 1,
              author: searchAuthorFromPost($post),
              text: text,
              lowerText: text.toLocaleLowerCase()
            });
          });
          searchPageCache[pageNumber] = { html: html, items: items };
          return items;
        }
        window.__SNS_HISTORY_SEARCH_INDEX__ = function() {
          // Keep chats that never use search free of this observer.
          ensureSearchObserver();
          // Flush synchronous edits before MutationObserver's async callback.
          if (searchObserver) invalidateSearchMutations(searchObserver.takeRecords());
          var result = [];
          var positions = Object.create(null);
          var sourcePages = Object.create(null);
          Object.keys(searchPageCache).forEach(function(pageNumber) {
            if (!Object.prototype.hasOwnProperty.call(pageResults, pageNumber)) delete searchPageCache[pageNumber];
          });
          Object.keys(pageResults).sort(function(a, b) {
            return Number(a) - Number(b);
          }).forEach(function(pageNumber) {
            var items = cachedPageSearchItems(pageNumber);
            items.forEach(function(item) {
              var key = item.postId;
              if (Object.prototype.hasOwnProperty.call(positions, key)) return;
              positions[key] = result.length;
              sourcePages[key] = Number(pageNumber) || 1;
              result.push(item);
            });
          });
          $topic.children(".post.sns-message").each(function() {
            var item = cachedLiveSearchItem(this);
            if (!item) return;
            var key = item.postId;
            var sourcePage = sourcePages[key];
            var liveItem = {
              postId: key,
              page: sourcePage || Number($(this).attr("data-sns-source-page")) || maxPage,
              author: item.author,
              text: item.text,
              lowerText: item.lowerText
            };
            if (Object.prototype.hasOwnProperty.call(positions, key)) {
              result[positions[key]] = liveItem;
            } else {
              positions[key] = result.length;
              result.push(liveItem);
            }
          });
          return result.filter(function(item) { return !!item.text; });
        };
        function clearSearchTempHistory() {
          var changed = false;
          $topic.children(".sns-search-temp").each(function() {
            var $post = $(this);
            var sourcePage = Number($post.attr("data-sns-source-page") || 0);
            if (sourcePage && lazyOldestLoadedPage !== null && sourcePage >= lazyOldestLoadedPage) {
              $post.removeClass("sns-search-temp").removeAttr("data-sns-source-page");
              return;
            }
            $post.remove();
            changed = true;
          });
          if (changed) {
            $topic.children(".sns-story-date-separator, .sns-date-separator").remove();
            $(document).trigger("sns_pages_loaded");
          }
        }
        window.__SNS_HISTORY_CLEAR_SEARCH_TEMP__ = clearSearchTempHistory;
        window.__SNS_HISTORY_SHOW_SEARCH_POST__ = function(pageNumber, postId) {
          pageNumber = Number(pageNumber) || 1;
          postId = String(postId || "");
          if (!postId) {
            return false;
          }
          clearSearchTempHistory();
          var $existing = $topic.children("#" + postId).first();
          if (!$existing.length) {
            var html = pageResults[pageNumber];
            if (!html) {
              return false;
            }
            var before = {};
            $topic.children(".post").each(function() {
              var key = postKey($(this));
              if (key) {
                before[key] = true;
              }
            });
            appendPostsFromHtml($topic, html);
            $topic.children(".post").each(function() {
              var $post = $(this);
              var key = postKey($post);
              if (key && !before[key]) {
                $post.addClass("sns-search-temp").attr("data-sns-source-page", String(pageNumber));
              }
            });
            sortTopicPostsChronologically($topic);
            $(document).trigger("sns_pages_loaded");
            $existing = $topic.children("#" + postId).first();
          }
          if (!$existing.length) {
            return false;
          }
          var topicNode = $topic.get(0);
          if (!topicNode) {
            return false;
          }
          setTimeout(function() {
            var topicRect = topicNode.getBoundingClientRect();
            var postRect = $existing.get(0).getBoundingClientRect();
            var destination = topicNode.scrollTop + postRect.top - topicRect.top - Math.max(70, topicNode.clientHeight * .22);
            topicNode.scrollTo({
              top: Math.max(0, destination),
              behavior: "smooth"
            });
            $topic.children(".sns-search-target").removeClass("sns-search-target");
            $existing.addClass("sns-search-target");
            setTimeout(function() {
              $existing.removeClass("sns-search-target");
            }, 1900);
          }, 70);
          return true;
        };
        maxPage = Math.min(100, Math.max(1, maxTopicPage()));
        window.__SNS_PHYSICAL_HISTORY_MAX_PAGE__ = maxPage;
        function pinInitialBottom() {
          var topicNode = $topic.get(0);
          if (!topicNode) {
            return;
          }
          var cancelled = false;
          function cancelPin() {
            cancelled = true;
          }
          function goBottom() {
            if (!cancelled && topicNode) {
              topicNode.scrollTop = topicNode.scrollHeight;
            }
          }
          $topic.one("wheel.snsInitialBottom touchstart.snsInitialBottom pointerdown.snsInitialBottom", cancelPin);
          [ 0, 70, 190, 430, 850 ].forEach(function(delay) {
            setTimeout(goBottom, delay);
          });
          setTimeout(function() {
            $topic.off(".snsInitialBottom");
          }, 1100);
        }
        function markInitialReady() {
          if (!window.__SNS_INITIAL_HISTORY_READY__) {
            window.__SNS_INITIAL_HISTORY_READY__ = true;
            $(document).trigger("sns_initial_history_ready");
          }
        }
        function finishNewestPages(firstHtml, lastHtml, firstPage) {
          if (firstHtml) {
            pageResults[firstPage] = firstHtml;
          }
          if (lastHtml && maxPage !== firstPage) {
            pageResults[maxPage] = lastHtml;
          }
          removeDisplayedNonConfigPosts();
          if (firstHtml) {
            appendPostsFromHtml($topic, firstHtml);
          }
          if (lastHtml && maxPage !== firstPage) {
            appendPostsFromHtml($topic, lastHtml);
          }
          sortTopicPostsChronologically($topic);
          lazyOldestLoadedPage = firstPage;
          window.__SNS_LAZY_HISTORY_ENABLED__ = true;
          window.__SNS_LAZY_HISTORY_MAX_PAGE__ = maxPage;
          window.__SNS_LAZY_HISTORY_OLDEST_PAGE__ = lazyOldestLoadedPage;
          updateHistoryLoader();
          installLazyHistoryControls();
          $("#sns-pages-loading").stop(true, true).remove();
          $("#sns-empty").remove();
          $(document).trigger("sns_pages_loaded");
          markInitialReady();
          pinInitialBottom();
        }
        if ($loading.length) {
          $loading.text("\u0437\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u043f\u043e\u0441\u043b\u0435\u0434\u043d\u0438\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f\u2026");
        }
        if (maxPage <= 1) {
          lazyOldestLoadedPage = 1;
          window.__SNS_LAZY_HISTORY_ENABLED__ = true;
          window.__SNS_LAZY_HISTORY_MAX_PAGE__ = 1;
          window.__SNS_LAZY_HISTORY_OLDEST_PAGE__ = 1;
          updateHistoryLoader();
          installLazyHistoryControls();
          $("#sns-pages-loading").remove();
          $(document).trigger("sns_pages_loaded");
          markInitialReady();
          pinInitialBottom();
          return;
        }
        var firstRecentPage = Math.max(1, maxPage - 1);
        var firstRequest = fetchPage(firstRecentPage, 1);
        var lastRequest = maxPage === firstRecentPage ? firstRequest : fetchPage(maxPage, 1);
        $.when(firstRequest, lastRequest).then(function(firstResult, lastResult) {
          var firstHtml = Array.isArray(firstResult) ? firstResult[0] : firstResult;
          var lastHtml = Array.isArray(lastResult) ? lastResult[0] : lastResult;
          finishNewestPages(firstHtml, lastHtml, firstRecentPage);
        }, function() {
          fetchPage(maxPage, 1).then(function(lastHtml) {
            finishNewestPages(lastHtml, null, maxPage);
          }, function() {
            $("#sns-pages-loading").remove();
            lazyOldestLoadedPage = maxPage;
            window.__SNS_LAZY_HISTORY_ENABLED__ = true;
            window.__SNS_LAZY_HISTORY_MAX_PAGE__ = maxPage;
            window.__SNS_LAZY_HISTORY_OLDEST_PAGE__ = lazyOldestLoadedPage;
            updateHistoryLoader();
            installLazyHistoryControls();
            markInitialReady();
            pinInitialBottom();
          });
        });
      }
      function install() {
        if (!document.body.classList.contains("sns-chat-page") || !$("#sns-chat-shell>.topic").length) {
          return false;
        }
        if (window.__SNS_ALL_PAGES_SYNC_STARTED__) {
          return true;
        }
        window.__SNS_ALL_PAGES_SYNC_STARTED__ = true;
        installPublishedNoticeSuppressor();
        window.SNSHidePublishedNotice = hidePublishedRusffNotice;
        setTimeout(syncPagedHistoryFallback, 100);
        setTimeout(function() {
          $("#sns-chat-shell>.topic").find('.post[data-sns-api-post="1"]').each(function() {
            var $post = $(this);
            var userId = String($post.attr("data-user-id") || "");
            var username = cleanText($post.find(".pa-author a").first().text());
            if (!userId || typeof window.SNSResolveUserVisual !== "function") {
              return;
            }
            window.SNSResolveUserVisual(userId, username, function(data) {
              if (!data || !data.avatar) {
                return;
              }
              var $author = $post.find(".post-author ul").first();
              if (!$author.length) {
                return;
              }
              var $avatar = $author.find(".pa-avatar").first();
              if (!$avatar.length) {
                $avatar = $('<li class="pa-avatar"></li>').appendTo($author);
              }
              $avatar.empty().append($('<img alt="">').attr("src", data.avatar));
              if (data.name) {
                $post.find(".pa-author a").first().text(data.name);
              }
              $(document).trigger("sns_user_visual_loaded");
            });
          });
        }, 900);
        return true;
      }
      var attempts = 0;
      var timer = setInterval(function() {
        attempts++;
        if (install() || attempts > 80) {
          clearInterval(timer);
        }
      }, 100);
    })(jQuery);
    (function() {
      "use strict";
      var APP_ID = 16777215;
      var META_KEY = "res_sm_meta_v1";
      var CAT_PREFIX = "res_sm_cat_";
      var PERSONAL_KEY = "res_sm_personal_v1";
      var pickerState = {
        open: false,
        loading: false,
        categories: [],
        activeId: "",
        lastLoadAt: 0
      };
      function q(selector, root) {
        return (root || document).querySelector(selector);
      }
      function qa(selector, root) {
        return Array.prototype.slice.call((root || document).querySelectorAll(selector));
      }
      function uniqUrls(list) {
        var seen = {};
        return (list || []).map(function(item) {
          return String(item || "").trim();
        }).filter(function(url) {
          if (!/^https?:\/\/\S+$/i.test(url) || seen[url]) {
            return false;
          }
          seen[url] = true;
          return true;
        });
      }
      function parseJson(value, fallback) {
        try {
          return value ? JSON.parse(value) : fallback;
        } catch (error) {
          return fallback;
        }
      }
      function personalUrls() {
        try {
          return uniqUrls(JSON.parse(localStorage.getItem(PERSONAL_KEY) || "[]"));
        } catch (error) {
          return [];
        }
      }
      function storageGet(keys) {
        return new Promise(function(resolve, reject) {
          SNSRequest({
            url: "/api.php",
            type: "GET",
            dataType: "json",
            data: {
              method: "storage.get",
              app_id: APP_ID,
              key: keys
            },
            timeout: 1e4
          }).done(function(response) {
            if (response && response.error) {
              reject(new Error("storage.get failed"));
              return;
            }
            resolve(response && response.response && response.response.storage ? response.response.storage.data || {} : {});
          }).fail(function(xhr) {
            reject(new Error("storage.get " + (xhr.status || "")));
          });
        });
      }
      function scrapeRenderedForumPanel() {
        var host = q("#smilies-area");
        if (!host) {
          return [];
        }
        var tabs = qa(".tabs li.resGen", host).filter(function(li) {
          var name = String(li.textContent || "").trim();
          return name && !/^(\u0421\u0412\u041e\u0418|\u0423\u041f\u0420\u0410\u0412\u041b\u0415\u041d\u0418\u0415)$/i.test(name);
        });
        var panes = qa("#wrapper > .resPane.resGen", host).filter(function(pane) {
          return !pane.classList.contains("t-personal") && !pane.classList.contains("t-manager");
        });
        var result = [];
        tabs.forEach(function(li, index) {
          var pane = panes[index];
          if (!pane) {
            return;
          }
          var urls = uniqUrls(qa("img", pane).map(function(img) {
            return img.getAttribute("src") || img.getAttribute("data-src") || "";
          }));
          result.push({
            id: "dom_" + index,
            name: String(li.textContent || "").trim(),
            urls: urls,
            personal: false
          });
        });
        return result;
      }
      async function loadCategories() {
        var categories = [];
        try {
          var metaData = await storageGet(META_KEY);
          var meta = parseJson(metaData[META_KEY], null);
          if (meta && Array.isArray(meta.categories)) {
            var sharedMeta = meta.categories.filter(function(category) {
              return category && category.id && category.name;
            });
            var keys = sharedMeta.map(function(category) {
              return CAT_PREFIX + category.id;
            });
            var catData = keys.length ? await storageGet(keys) : {};
            categories = sharedMeta.map(function(category) {
              return {
                id: String(category.id),
                name: String(category.name),
                urls: uniqUrls(parseJson(catData[CAT_PREFIX + category.id], [])),
                personal: false
              };
            });
          }
        } catch (error) {
          console.warn("[SNS smilies] shared storage read failed", error);
        }
        if (!categories.length) {
          categories = scrapeRenderedForumPanel();
        }
        categories.push({
          id: "__personal__",
          name: "\u0421\u0412\u041e\u0418",
          urls: personalUrls(),
          personal: true
        });
        return categories;
      }
      function ensureStyles() {
        if (q("#sns-smilies-picker-style")) {
          return;
        }
        var style = document.createElement("style");
        style.id = "sns-smilies-picker-style";
        style.textContent = [ "body.sns-chat-page #sns-composer-ui>.sns-ui-input-wrap{position:relative!important;}", "body.sns-chat-page #sns-ui-input{padding-right:44px!important;}", "body.sns-chat-page .sns-ui-smilies{", "position:absolute!important;right:5px!important;bottom:4px!important;z-index:6!important;", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:34px!important;height:34px!important;margin:0!important;padding:0!important;", "border:0!important;border-radius:50%!important;background:transparent!important;", "color:#8b8b94!important;cursor:pointer!important;", 'font:400 18px/34px "Apple Color Emoji","Segoe UI Emoji","Noto Color Emoji",sans-serif!important;', "transition:background .15s ease,color .15s ease,transform .15s ease!important;", "}", "body.sns-chat-page .sns-ui-smilies:hover,body.sns-chat-page .sns-ui-smilies.is-active{", "background:rgba(0,0,0,.055)!important;color:var(--sns-own-g1,#b65f3a)!important;", "}", "body.sns-chat-page .sns-ui-smilies:active{transform:scale(.94)!important;}", "body.sns-chat-page.sns-chat-readonly .sns-ui-smilies{display:none!important;}", "#sns-smilies-picker{", "position:fixed!important;z-index:2147483200!important;display:none!important;", "box-sizing:border-box!important;width:min(430px,calc(100vw - 20px))!important;", "height:292px!important;margin:0!important;padding:0!important;", "overflow:hidden!important;background:rgba(248,248,250,.985)!important;", "border:1px solid rgba(35,35,42,.10)!important;border-radius:14px!important;", "box-shadow:0 12px 34px rgba(20,22,30,.16),0 28px 64px rgba(20,22,30,.15)!important;", "backdrop-filter:blur(12px)!important;-webkit-backdrop-filter:blur(12px)!important;", "color:#3f3f46!important;", "font:11px/1.3 Arial,sans-serif!important;", "}", "#sns-smilies-picker.is-open{display:flex!important;flex-direction:column!important;}", "#sns-smilies-picker .sns-smilie-tabs{", "display:flex!important;align-items:center!important;gap:3px!important;", "flex:0 0 42px!important;box-sizing:border-box!important;", "width:100%!important;margin:0!important;padding:6px 7px!important;", "overflow-x:auto!important;overflow-y:hidden!important;", "border-bottom:1px solid rgba(0,0,0,.065)!important;", "scrollbar-width:none!important;", "}", "#sns-smilies-picker .sns-smilie-tabs::-webkit-scrollbar{display:none!important;}", "#sns-smilies-picker .sns-smilie-tab{", "flex:0 0 auto!important;height:29px!important;margin:0!important;padding:0 10px!important;", "border:0!important;border-radius:8px!important;background:transparent!important;", "color:#777780!important;cursor:pointer!important;", "font:700 9px/29px Arial,sans-serif!important;text-transform:uppercase!important;", "white-space:nowrap!important;", "}", "#sns-smilies-picker .sns-smilie-tab:hover{background:rgba(0,0,0,.045)!important;color:#45454d!important;}", "#sns-smilies-picker .sns-smilie-tab.is-active{", "background:color-mix(in srgb,var(--sns-own-g1,#b65f3a) 11%,transparent)!important;", "color:var(--sns-own-g1,#b65f3a)!important;", "}", "#sns-smilies-picker .sns-smilie-body{", "position:relative!important;flex:1 1 auto!important;min-height:0!important;", "box-sizing:border-box!important;width:100%!important;padding:9px!important;", "overflow-y:auto!important;overflow-x:hidden!important;", "}", "#sns-smilies-picker .sns-smilie-grid{", "display:grid!important;grid-template-columns:repeat(auto-fill,minmax(56px,1fr))!important;", "align-items:center!important;gap:5px!important;width:100%!important;", "}", "#sns-smilies-picker .sns-smilie-item{", "display:flex!important;align-items:center!important;justify-content:center!important;", "box-sizing:border-box!important;min-width:0!important;height:62px!important;", "margin:0!important;padding:5px!important;border:0!important;border-radius:9px!important;", "background:transparent!important;cursor:pointer!important;", "transition:background .12s ease,transform .12s ease!important;", "}", "#sns-smilies-picker .sns-smilie-item:hover{background:rgba(0,0,0,.045)!important;transform:translateY(-1px)!important;}", "#sns-smilies-picker .sns-smilie-item img{", "display:block!important;max-width:100%!important;max-height:52px!important;", "width:auto!important;height:auto!important;margin:0!important;padding:0!important;", "border:0!important;border-radius:0!important;object-fit:contain!important;", "pointer-events:none!important;", "}", "#sns-smilies-picker .sns-smilie-empty,#sns-smilies-picker .sns-smilie-loading{", "display:flex!important;align-items:center!important;justify-content:center!important;", "box-sizing:border-box!important;width:100%!important;height:100%!important;", "padding:24px!important;color:#8a8a92!important;", "font:10px/1.5 Arial,sans-serif!important;text-align:center!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message .post-content img.smalimg,", "body.sns-chat-page #pun-viewtopic .sns-message .post-content img.sns-inline-smilie,", 'body.sns-chat-page #pun-viewtopic .sns-message .post-content img[alt="smalimg"],', 'body.sns-chat-page #pun-viewtopic .sns-message .post-content img[title="smalimg"]{', "display:inline-block!important;vertical-align:middle!important;", "width:auto!important;height:auto!important;", "max-width:130px!important;max-height:130px!important;", "margin:1px 2px!important;padding:0!important;", "border:0!important;border-radius:0!important;object-fit:contain!important;", "cursor:default!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-with-text .post-content img.smalimg,", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-with-text .post-content img.sns-inline-smilie,", 'body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-with-text .post-content img[alt="smalimg"],', 'body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-with-text .post-content img[title="smalimg"]{', "display:inline-block!important;vertical-align:middle!important;", "max-width:76px!important;max-height:76px!important;", "margin:0 3px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-with-text .post-content p{", "display:block!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only .post-body{", "width:fit-content!important;min-width:0!important;max-width:none!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only .post-box{", "width:fit-content!important;min-width:0!important;max-width:none!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only .post-content{", "width:fit-content!important;min-width:0!important;max-width:none!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only .post-content,", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only.sns-own .post-content,", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only.sns-other .post-content{", "background:transparent!important;", "background-image:none!important;", "padding:0!important;", "border:0!important;border-radius:0!important;", "box-shadow:none!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only .post-content p{", "margin:0!important;padding:0!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only .post-content img.smalimg,", "body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only .post-content img.sns-inline-smilie,", 'body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only .post-content img[alt="smalimg"],', 'body.sns-chat-page #pun-viewtopic .sns-message.sns-sticker-only .post-content img[title="smalimg"]{', "display:block!important;", "max-width:155px!important;max-height:155px!important;", "margin:0!important;", "}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.sns-pending-sticker-text .sns-pending-bubble{", "white-space:normal!important;", "}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.sns-pending-sticker-text .sns-pending-sticker{", "display:inline-block!important;vertical-align:middle!important;", "width:auto!important;height:auto!important;", "max-width:76px!important;max-height:76px!important;", "margin:0 3px!important;border:0!important;border-radius:0!important;", "object-fit:contain!important;", "}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.sns-pending-sticker-only .sns-pending-bubble{", "background:transparent!important;background-image:none!important;", "padding:0!important;border:0!important;border-radius:0!important;", "box-shadow:none!important;", "}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.sns-pending-sticker-only .sns-pending-sticker{", "display:block!important;width:auto!important;height:auto!important;", "max-width:155px!important;max-height:155px!important;", "margin:0!important;border:0!important;border-radius:0!important;", "object-fit:contain!important;", "}", "@media(max-width:650px){", "#sns-smilies-picker{width:calc(100vw - 16px)!important;height:276px!important;}", "#sns-smilies-picker .sns-smilie-grid{grid-template-columns:repeat(auto-fill,minmax(52px,1fr))!important;}", "#sns-smilies-picker .sns-smilie-item{height:58px!important;padding:4px!important;}", "#sns-smilies-picker .sns-smilie-item img{max-height:49px!important;}", "}" ].join("");
        document.head.appendChild(style);
      }
      function ensurePicker() {
        var picker = q("#sns-smilies-picker");
        if (picker) {
          return picker;
        }
        picker = document.createElement("div");
        picker.id = "sns-smilies-picker";
        picker.setAttribute("aria-hidden", "true");
        picker.innerHTML = '<div class="sns-smilie-tabs"></div>' + '<div class="sns-smilie-body">' + '<div class="sns-smilie-loading">\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u0441\u043c\u0430\u0439\u043b\u0438\u043a\u043e\u0432\u2026</div>' + "</div>";
        document.body.appendChild(picker);
        return picker;
      }
      function button() {
        return q(".sns-ui-smilies");
      }
      function input() {
        return q("#sns-ui-input");
      }
      function closePicker() {
        if (pickerState.imageObserver) {
          pickerState.imageObserver.disconnect();
          pickerState.imageObserver = null;
        }
        var picker = q("#sns-smilies-picker");
        var btn = button();
        pickerState.open = false;
        if (picker) {
          picker.classList.remove("is-open");
          picker.setAttribute("aria-hidden", "true");
        }
        if (picker) {
          var grid = q(".sns-smilie-body", picker);
          if (grid) grid.innerHTML = "";
        }
        if (btn) {
          btn.classList.remove("is-active");
          btn.setAttribute("aria-expanded", "false");
        }
      }
      function positionPicker() {
        if (!pickerState.open) {
          return;
        }
        var picker = ensurePicker();
        var btn = button();
        if (!btn) {
          return;
        }
        var rect = btn.getBoundingClientRect();
        var width = picker.offsetWidth || Math.min(430, window.innerWidth - 20);
        var height = picker.offsetHeight || 292;
        var left = rect.right - width;
        left = Math.max(8, Math.min(left, window.innerWidth - width - 8));
        var top = rect.top - height - 9;
        if (top < 8) {
          top = Math.min(window.innerHeight - height - 8, rect.bottom + 9);
        }
        picker.style.left = Math.round(left) + "px";
        picker.style.top = Math.round(Math.max(8, top)) + "px";
      }
      function insertAtCaret(url) {
        var field = input();
        if (!field) {
          return;
        }
        var token = "[img=smalimg]" + url + "[/img]";
        var value = String(field.value || "");
        var start = typeof field.selectionStart === "number" ? field.selectionStart : value.length;
        var end = typeof field.selectionEnd === "number" ? field.selectionEnd : start;
        var before = value.slice(0, start);
        var after = value.slice(end);
        var prefix = before && !/[\s\n]$/.test(before) ? " " : "";
        var suffix = after && !/^[\s\n]/.test(after) ? " " : "";
        field.value = before + prefix + token + suffix + after;
        var caret = (before + prefix + token + suffix).length;
        try {
          field.setSelectionRange(caret, caret);
        } catch (error) {}
        field.dispatchEvent(new Event("input", {
          bubbles: true
        }));
        field.focus();
      }
      function renderActiveCategory() {
        var picker = ensurePicker();
        var body = q(".sns-smilie-body", picker);
        if (!body) return;
        if (pickerState.imageObserver) {
          pickerState.imageObserver.disconnect();
          pickerState.imageObserver = null;
        }
        body.innerHTML = "";
        var category = pickerState.categories.find(function(item) {
          return item.id === pickerState.activeId;
        });
        if (!category || !category.urls.length) {
          var empty = document.createElement("div");
          empty.className = "sns-smilie-empty";
          empty.textContent = category && category.personal ? "\u0412 \xab\u0421\u0412\u041e\u0418\xbb \u043f\u043e\u043a\u0430 \u043d\u0438\u0447\u0435\u0433\u043e \u043d\u0435\u0442. \u0414\u043e\u0431\u0430\u0432\u0438\u0442\u044c \u043c\u043e\u0436\u043d\u043e \u0447\u0435\u0440\u0435\u0437 \u043e\u0431\u044b\u0447\u043d\u0443\u044e \u0444\u043e\u0440\u0443\u043c\u043d\u0443\u044e \u043f\u0430\u043d\u0435\u043b\u044c \u0441\u043c\u0430\u0439\u043b\u0438\u043a\u043e\u0432." : "\u0412 \u044d\u0442\u043e\u0439 \u043a\u0430\u0442\u0435\u0433\u043e\u0440\u0438\u0438 \u043f\u043e\u043a\u0430 \u043d\u0438\u0447\u0435\u0433\u043e \u043d\u0435\u0442.";
          body.appendChild(empty);
          return;
        }
        var observer = null;
        if (window.IntersectionObserver) {
          observer = new IntersectionObserver(function(entries) {
            entries.forEach(function(entry) {
              if (entry.isIntersecting) {
                var image = entry.target;
                image.src = image.getAttribute("data-sns-src");
                image.removeAttribute("data-sns-src");
                observer.unobserve(image);
              }
            });
          }, {
            root: body,
            rootMargin: "60px"
          });
          pickerState.imageObserver = observer;
        }
        var grid = document.createElement("div");
        grid.className = "sns-smilie-grid";
        var more = document.createElement("button");
        more.type = "button";
        more.className = "sns-smilie-more";
        more.textContent = "\u041f\u043e\u043a\u0430\u0437\u0430\u0442\u044c \u0435\u0449\u0435";
        var shown = 0;
        function appendBatch() {
          category.urls.slice(shown, shown + 48).forEach(function(url) {
            var item = document.createElement("button");
            item.type = "button";
            item.className = "sns-smilie-item";
            item.title = "\u0412\u0441\u0442\u0430\u0432\u0438\u0442\u044c";
            var image = document.createElement("img");
            image.loading = "lazy";
            image.decoding = "async";
            image.alt = "";
            if (observer) image.setAttribute("data-sns-src", url); else image.src = url;
            item.appendChild(image);
            item.addEventListener("click", function(event) {
              event.preventDefault();
              event.stopPropagation();
              insertAtCaret(url);
            });
            grid.appendChild(item);
            if (observer) observer.observe(image);
          });
          shown += 48;
          more.hidden = shown >= category.urls.length;
        }
        body.appendChild(grid);
        body.appendChild(more);
        more.addEventListener("click", appendBatch);
        appendBatch();
      }
      function renderCategories() {
        var picker = ensurePicker();
        var tabs = q(".sns-smilie-tabs", picker);
        tabs.innerHTML = "";
        if (!pickerState.categories.length) {
          q(".sns-smilie-body", picker).innerHTML = '<div class="sns-smilie-empty">\u041a\u0430\u0442\u0435\u0433\u043e\u0440\u0438\u0438 \u043d\u0435 \u043d\u0430\u0439\u0434\u0435\u043d\u044b.</div>';
          return;
        }
        var activeStillExists = pickerState.categories.some(function(category) {
          return category.id === pickerState.activeId;
        });
        if (!activeStillExists) {
          pickerState.activeId = pickerState.categories[0].id;
        }
        pickerState.categories.forEach(function(category) {
          var tab = document.createElement("button");
          tab.type = "button";
          tab.className = "sns-smilie-tab";
          if (category.id === pickerState.activeId) {
            tab.classList.add("is-active");
          }
          tab.textContent = category.name;
          tab.addEventListener("click", function(event) {
            event.preventDefault();
            event.stopPropagation();
            pickerState.activeId = category.id;
            qa(".sns-smilie-tab", tabs).forEach(function(node) {
              node.classList.remove("is-active");
            });
            tab.classList.add("is-active");
            renderActiveCategory();
          });
          tabs.appendChild(tab);
        });
        renderActiveCategory();
      }
      async function refreshPicker() {
        if (pickerState.loading) {
          return;
        }
        pickerState.loading = true;
        var picker = ensurePicker();
        var body = q(".sns-smilie-body", picker);
        if (body) {
          body.innerHTML = '<div class="sns-smilie-loading">\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u0441\u043c\u0430\u0439\u043b\u0438\u043a\u043e\u0432\u2026</div>';
        }
        try {
          pickerState.categories = await loadCategories();
          pickerState.lastLoadAt = Date.now();
          if (pickerState.open) renderCategories();
        } finally {
          pickerState.loading = false;
          requestAnimationFrame(positionPicker);
        }
      }
      function openPicker() {
        var picker = ensurePicker();
        var btn = button();
        if (!btn) {
          return;
        }
        var attachMenu = q("#sns-attach-menu");
        if (attachMenu) {
          attachMenu.classList.remove("is-open");
        }
        var formatToolbar = q("#sns-format-toolbar");
        if (formatToolbar) {
          formatToolbar.classList.remove("is-open");
          formatToolbar.setAttribute("aria-hidden", "true");
        }
        var formatButton = q(".sns-ui-format");
        if (formatButton) {
          formatButton.classList.remove("is-active");
          formatButton.setAttribute("aria-expanded", "false");
        }
        pickerState.open = true;
        picker.classList.add("is-open");
        picker.setAttribute("aria-hidden", "false");
        btn.classList.add("is-active");
        btn.setAttribute("aria-expanded", "true");
        positionPicker();
        if (pickerState.categories.length && Date.now() - pickerState.lastLoadAt < 6e4) {
          pickerState.categories.forEach(function(category) {
            if (category.personal) category.urls = personalUrls();
          });
          renderCategories();
        } else refreshPicker();
      }
      function togglePicker() {
        if (pickerState.open) {
          closePicker();
        } else {
          openPicker();
        }
      }
      function installButton() {
        var wrap = q("#sns-composer-ui > .sns-ui-input-wrap");
        if (!wrap) {
          return false;
        }
        if (q(".sns-ui-smilies", wrap)) {
          return true;
        }
        var btn = document.createElement("button");
        btn.type = "button";
        btn.className = "sns-ui-smilies";
        btn.title = "\u0421\u043c\u0430\u0439\u043b\u0438\u043a\u0438 \u0438 \u0441\u0442\u0438\u043a\u0435\u0440\u044b";
        btn.setAttribute("aria-label", "\u0421\u043c\u0430\u0439\u043b\u0438\u043a\u0438 \u0438 \u0441\u0442\u0438\u043a\u0435\u0440\u044b");
        btn.setAttribute("aria-expanded", "false");
        btn.textContent = String.fromCharCode(9786);
        btn.addEventListener("click", function(event) {
          event.preventDefault();
          event.stopPropagation();
          togglePicker();
        });
        wrap.appendChild(btn);
        return true;
      }
      function init() {
        ensureStyles();
        ensurePicker();
        if (installButton()) {
          return;
        }
        var attempts = 0;
        var timer = setInterval(function() {
          attempts += 1;
          if (installButton() || attempts >= 40) {
            clearInterval(timer);
          }
        }, 250);
      }
      document.addEventListener("click", function(event) {
        if (!pickerState.open) {
          return;
        }
        var picker = q("#sns-smilies-picker");
        var btn = button();
        if (picker && picker.contains(event.target)) {
          return;
        }
        if (btn && (event.target === btn || btn.contains(event.target))) {
          return;
        }
        closePicker();
      }, false);
      document.addEventListener("keydown", function(event) {
        if (event.key === "Escape" && pickerState.open) {
          closePicker();
        }
      }, false);
      document.addEventListener("click", function(event) {
        if (event.target.closest && event.target.closest(".sns-ui-plus, .sns-ui-format, .sns-ui-send")) {
          closePicker();
        }
      }, false);
      window.addEventListener("resize", function() {
        if (pickerState.open) {
          positionPicker();
        }
      });
      if (document.readyState === "loading") {
        document.addEventListener("DOMContentLoaded", init);
      } else {
        init();
      }
    })();
    (function() {
      "use strict";
      if (document.getElementById("sns-story-date-time-style-v191")) {
        return;
      }
      var style = document.createElement("style");
      style.id = "sns-story-date-time-style-v191";
      style.textContent = [ "body.sns-chat-page #sns-composer-ui{", "grid-template-columns:30px 30px 30px minmax(0,1fr) 38px!important;", "}", "body.sns-chat-page #sns-composer-ui>.sns-ui-plus{grid-column:1!important;}", "body.sns-chat-page #sns-composer-ui>.sns-ui-format{grid-column:2!important;}", "body.sns-chat-page #sns-composer-ui>.sns-ui-storytime{grid-column:3!important;}", "body.sns-chat-page #sns-composer-ui>.sns-ui-input-wrap{grid-column:4!important;}", "body.sns-chat-page #sns-composer-ui>.sns-ui-send{grid-column:5!important;}", "body.sns-chat-page .sns-ui-storytime{", "grid-row:2!important;", "align-self:center!important;", "display:flex!important;align-items:center!important;justify-content:center!important;", "box-sizing:border-box!important;", "width:30px!important;height:30px!important;min-width:30px!important;", "margin:0!important;padding:0!important;", "border:0!important;border-radius:50%!important;", "background:transparent!important;color:#8c8d94!important;", "cursor:pointer!important;", "position:relative!important;top:-1px!important;", "line-height:0!important;", "transition:background .15s ease,color .15s ease,transform .15s ease!important;", "}", "body.sns-chat-page .sns-ui-storytime:hover,body.sns-chat-page .sns-ui-storytime.is-active{", "background:rgba(0,0,0,.055)!important;", "color:var(--sns-own-g1,#b65f3a)!important;", "}", "body.sns-chat-page .sns-ui-storytime:active{transform:scale(.94)!important;}", "body.sns-chat-page .sns-ui-storytime svg{", "display:block!important;width:15px!important;height:15px!important;", "margin:0!important;padding:0!important;", "position:relative!important;top:0!important;", "fill:none!important;stroke:currentColor!important;stroke-width:1.75!important;", "stroke-linecap:round!important;stroke-linejoin:round!important;", "}", "body.sns-chat-page #sns-story-compose{", "position:absolute!important;left:70px!important;bottom:64px!important;z-index:2760!important;", "display:none!important;", "box-sizing:border-box!important;width:320px!important;max-width:calc(100% - 88px)!important;", "padding:11px!important;", "background:rgba(255,255,255,.985)!important;color:#333!important;", "border:1px solid rgba(0,0,0,.08)!important;border-radius:12px!important;", "box-shadow:0 12px 32px rgba(18,20,28,.16),0 22px 52px rgba(18,20,28,.11)!important;", "}", "body.sns-chat-page #sns-story-compose.is-open{display:block!important;}", "body.sns-chat-page .sns-story-compose-title{", "display:flex!important;align-items:center!important;justify-content:space-between!important;", "gap:10px!important;margin:0 0 9px!important;", "font:700 11px/1.2 Arial,sans-serif!important;color:#444!important;", "}", "body.sns-chat-page .sns-story-compose-close{", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:23px!important;height:23px!important;margin:0!important;padding:0!important;", "border:0!important;border-radius:50%!important;background:transparent!important;", "color:#888!important;cursor:pointer!important;font:18px/23px Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-story-compose-close:hover{background:rgba(0,0,0,.06)!important;color:#444!important;}", "body.sns-chat-page .sns-story-fields{", "display:grid!important;grid-template-columns:minmax(0,1fr) minmax(0,.72fr)!important;", "gap:7px!important;", "}", "body.sns-chat-page .sns-story-field{display:flex!important;flex-direction:column!important;gap:4px!important;min-width:0!important;}", "body.sns-chat-page .sns-story-field>span{color:#888!important;font:700 8px/1.2 Arial,sans-serif!important;}", "body.sns-chat-page .sns-story-field input{", "box-sizing:border-box!important;width:100%!important;height:34px!important;", "margin:0!important;padding:0 9px!important;", "background:#f6f6f7!important;color:#333!important;", "border:1px solid rgba(0,0,0,.08)!important;border-radius:8px!important;", "outline:none!important;font:11px/34px Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-story-field input:focus{", "border-color:color-mix(in srgb,var(--sns-own-g1,#b65f3a) 42%,transparent)!important;", "}", "body.sns-chat-page .sns-story-compose-hint{", "margin-top:7px!important;color:#97979d!important;", "font:8px/1.35 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-story-compose-actions{", "display:flex!important;align-items:center!important;justify-content:flex-end!important;", "gap:6px!important;margin-top:9px!important;", "}", "body.sns-chat-page .sns-story-compose-actions button{", "height:29px!important;margin:0!important;padding:0 10px!important;", "border:0!important;border-radius:7px!important;cursor:pointer!important;", "font:700 9px/29px Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-story-clear{background:transparent!important;color:#888!important;}", "body.sns-chat-page .sns-story-apply{background:var(--sns-own-g1,#b65f3a)!important;color:#fff!important;}", "body.sns-chat-page #pun-viewtopic .sns-story-date-separator{", "position:relative!important;clear:both!important;", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:100%!important;box-sizing:border-box!important;", "margin:18px 0 15px!important;padding:0!important;", "pointer-events:none!important;z-index:3!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-story-date-separator>span{", "display:inline-flex!important;align-items:center!important;justify-content:center!important;", "min-height:22px!important;box-sizing:border-box!important;", "padding:4px 11px!important;", "border:1px solid rgba(70,66,62,.10)!important;border-radius:999px!important;", "background:rgba(245,243,239,.78)!important;color:rgba(68,65,62,.68)!important;", "box-shadow:0 2px 8px rgba(27,25,23,.045)!important;", "backdrop-filter:blur(7px)!important;-webkit-backdrop-filter:blur(7px)!important;", "font:700 8px/1.2 Arial,sans-serif!important;", "letter-spacing:.45px!important;text-transform:uppercase!important;", "white-space:nowrap!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-story-time{white-space:nowrap!important;}", "@media(max-width:650px){", "body.sns-chat-page #sns-composer-ui{grid-template-columns:30px 30px 30px minmax(0,1fr) 36px!important;}", "body.sns-chat-page #sns-story-compose{left:8px!important;right:8px!important;bottom:61px!important;width:auto!important;max-width:none!important;}", "body.sns-chat-page .sns-story-fields{grid-template-columns:1fr!important;}", "body.sns-chat-page #pun-viewtopic .sns-story-date-separator{margin:15px 0 12px!important;}", "body.sns-chat-page #pun-viewtopic .sns-story-date-separator>span{min-height:20px!important;padding:4px 9px!important;font-size:7px!important;}", "}" ].join("");
      document.head.appendChild(style);
    })();
    (function($) {
      "use strict";
      var VIDEO_PREFIX = "SNSVIDEO:";
      window.SNS_VIDEO_UPLOAD_CONFIG = window.SNS_VIDEO_UPLOAD_CONFIG || {
        cloudName: window.SNS_AUDIO_UPLOAD_CONFIG && window.SNS_AUDIO_UPLOAD_CONFIG.cloudName || "",
        uploadPreset: window.SNS_AUDIO_UPLOAD_CONFIG && window.SNS_AUDIO_UPLOAD_CONFIG.uploadPreset || "",
        maxMb: 80
      };
      var renderTimer = null;
      var observer = null;
      var uploadXhr = null;
      var uploadBusy = false;
      var uiInstalled = false;
      function encodeText(value) {
        try {
          return btoa(unescape(encodeURIComponent(String(value || ""))));
        } catch (error) {
          return "";
        }
      }
      function decodeText(value) {
        try {
          return decodeURIComponent(escape(atob(String(value || ""))));
        } catch (error) {
          return null;
        }
      }
      function safeUrl(value) {
        var raw = String(value || "").trim();
        return /^https?:\/\//i.test(raw) ? raw : "";
      }
      function browserPlayableVideoUrl(value) {
        var raw = safeUrl(value);
        if (!raw) {
          return "";
        }
        if (/https?:\/\/res\.cloudinary\.com\//i.test(raw) && /\/video\/upload\//i.test(raw)) {
          raw = raw.replace(/\/video\/upload\/sp_auto\//i, "/video/upload/f_mp4,q_auto/");
          if (!/\/video\/upload\/f_mp4,q_auto\//i.test(raw) && /\.(?:mov|m4v|webm|ogv|m3u8)(?:\?|$)/i.test(raw)) {
            raw = raw.replace(/\/video\/upload\//i, "/video/upload/f_mp4,q_auto/");
          }
          raw = raw.replace(/\.(?:m3u8|mov|m4v|webm|ogv)(?=\?|$)/i, ".mp4");
        }
        return raw;
      }
      function normalizeData(data) {
        data = data && typeof data === "object" ? data : {};
        var url = browserPlayableVideoUrl(data.url);
        if (!url) {
          return null;
        }
        return {
          url: url,
          caption: String(data.caption || "").trim().slice(0, 3e3)
        };
      }
      function markerFromData(data) {
        var normalized = normalizeData(data);
        if (!normalized) {
          return "";
        }
        var encoded = encodeText(JSON.stringify(normalized));
        return encoded ? VIDEO_PREFIX + encoded : "";
      }
      function parseMarker(value) {
        var raw = String(value || "");
        var match = raw.match(/SNSVIDEO:([A-Za-z0-9+\/=]+)/);
        if (!match) {
          return null;
        }
        var decoded = decodeText(match[1]);
        if (decoded === null) {
          return null;
        }
        try {
          var data = normalizeData(JSON.parse(decoded));
          return data ? {
            encoded: match[1],
            data: data
          } : null;
        } catch (error) {
          return null;
        }
      }
      function createCard(data) {
        var normalized = normalizeData(data);
        if (!normalized) {
          return $("<div></div>");
        }
        var $card = $('<div class="sns-video-card">' + '<video class="sns-video-native" controls playsinline preload="none"></video>' + '<div class="sns-video-error-note">\u0412\u0438\u0434\u0435\u043e \u0432\u0440\u0435\u043c\u0435\u043d\u043d\u043e \u043d\u0435 \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u043b\u043e\u0441\u044c. \u041e\u0431\u043d\u043e\u0432\u0438\u0442\u0435 \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u0443 \u0438\u043b\u0438 \u043f\u043e\u043f\u0440\u043e\u0431\u0443\u0439\u0442\u0435 \u043f\u043e\u0437\u0436\u0435.</div>' + '<div class="sns-video-caption"></div>' + "</div>");
        var $video = $card.find(".sns-video-native");
        $video.attr("src", normalized.url);
        if (/\.mp4(?:\?|$)/i.test(normalized.url)) {
          $video.attr("type", "video/mp4");
        }
        $card.find(".sns-video-caption").text(normalized.caption);
        $video.off(".snsVideoStableV196").on("loadedmetadata.snsVideoStableV196 canplay.snsVideoStableV196", function() {
          $card.removeClass("is-video-error");
        }).on("error.snsVideoStableV196", function() {
          $card.addClass("is-video-error");
        });
        return $card;
      }
      function fitVideoCardToMedia($card) {
        if (!$card || !$card.length) {
          return;
        }
        var video = $card.find(".sns-video-native").get(0);
        if (!video) {
          return;
        }
        function applyFit() {
          var sourceW = Number(video.videoWidth) || 0;
          var sourceH = Number(video.videoHeight) || 0;
          if (sourceW <= 0 || sourceH <= 0) {
            return;
          }
          var mobile = window.matchMedia && window.matchMedia("(max-width:650px)").matches;
          var maxW = mobile ? Math.min(300, Math.max(180, Math.floor(window.innerWidth * .72))) : 340;
          var maxH = mobile ? Math.min(430, Math.max(260, Math.floor(window.innerHeight * .58))) : 430;
          var scale = Math.min(maxW / sourceW, maxH / sourceH);
          if (scale > 1) {
            scale = Math.min(scale, 3);
          }
          var width = Math.max(120, Math.round(sourceW * scale));
          var height = Math.max(120, Math.round(sourceH * scale));
          width = Math.min(width, maxW);
          height = Math.min(height, maxH);
          video.style.setProperty("width", width + "px", "important");
          video.style.setProperty("height", height + "px", "important");
          video.style.setProperty("max-width", width + "px", "important");
          video.style.setProperty("max-height", height + "px", "important");
          $card[0].style.setProperty("width", width + "px", "important");
          $card[0].style.setProperty("max-width", width + "px", "important");
          var $post = $card.closest(".sns-message");
          if ($post.length) {
            var bubbleW = width + 10;
            $post.find(".post-body,.post-box,.post-content").each(function() {
              this.style.setProperty("width", bubbleW + "px", "important");
              this.style.setProperty("min-width", bubbleW + "px", "important");
              this.style.setProperty("max-width", bubbleW + "px", "important");
            });
          }
          var $pending = $card.closest(".sns-pending-row");
          if ($pending.length) {
            var pendingW = width + 10;
            $pending.find(".sns-pending-bubble").each(function() {
              this.style.setProperty("width", pendingW + "px", "important");
              this.style.setProperty("max-width", pendingW + "px", "important");
            });
          }
        }
        if (video.readyState >= 1 && video.videoWidth && video.videoHeight) {
          applyFit();
        } else {
          $(video).off("loadedmetadata.snsVideoFitV195").one("loadedmetadata.snsVideoFitV195", applyFit);
        }
      }
      function renderRealVideos() {
        $("#pun-viewtopic .post.sns-message").each(function() {
          var $post = $(this);
          var $content = $post.find(".post-content").first();
          if (!$content.length) {
            return;
          }
          var $existingVideoCard = $content.children(".sns-video-card").first();
          if ($existingVideoCard.length) {
            fitVideoCardToMedia($existingVideoCard);
            return;
          }
          var payload = parseMarker($content.text());
          if (!payload || !payload.data) {
            return;
          }
          $post.addClass("sns-video-message").removeClass("sns-has-media sns-media-only sns-media-caption sns-image-only").attr("data-sns-video", payload.encoded).attr("data-sns-images", "0");
          var $videoCard = createCard(payload.data);
          $content.empty().append($videoCard);
          fitVideoCardToMedia($videoCard);
        });
      }
      function renderPendingVideos() {
        $("#sns-pending-stack .sns-pending-row").each(function() {
          var $row = $(this);
          var $bubble = $row.find(".sns-pending-bubble").first();
          if (!$bubble.length) {
            return;
          }
          var $existingPendingVideoCard = $bubble.children(".sns-video-card").first();
          if ($existingPendingVideoCard.length) {
            fitVideoCardToMedia($existingPendingVideoCard);
            return;
          }
          var payload = parseMarker($bubble.text());
          if (!payload || !payload.data) {
            return;
          }
          $row.addClass("sns-pending-video");
          var $pendingVideoCard = createCard(payload.data);
          $bubble.empty().append($pendingVideoCard);
          fitVideoCardToMedia($pendingVideoCard);
        });
      }
      function renderAll() {
        renderRealVideos();
        renderPendingVideos();
      }
      function scheduleRender(delay) {
        if (renderTimer) {
          clearTimeout(renderTimer);
        }
        renderTimer = setTimeout(function() {
          renderTimer = null;
          renderAll();
        }, delay === undefined ? 45 : delay);
      }
      function videoConfig() {
        var raw = window.SNS_VIDEO_UPLOAD_CONFIG && typeof window.SNS_VIDEO_UPLOAD_CONFIG === "object" ? window.SNS_VIDEO_UPLOAD_CONFIG : {};
        var maxMb = Number(raw.maxMb);
        if (!isFinite(maxMb) || maxMb <= 0) {
          maxMb = 80;
        }
        maxMb = Math.max(1, Math.min(100, maxMb));
        return {
          cloudName: String(raw.cloudName || "").trim(),
          uploadPreset: String(raw.uploadPreset || "").trim(),
          maxMb: maxMb
        };
      }
      function cloudinaryReady() {
        var cfg = videoConfig();
        return !!(cfg.cloudName && cfg.uploadPreset);
      }
      function fileAllowed(file) {
        if (!file) {
          return false;
        }
        var name = String(file.name || "");
        var mime = String(file.type || "");
        return /^video\//i.test(mime) || /\.(?:mp4|webm|mov|m4v|ogv)$/i.test(name);
      }
      function deliveryUrl(response) {
        var url = String(response && (response.secure_url || response.playback_url) || "").trim();
        if (!url) {
          return "";
        }
        return browserPlayableVideoUrl(url);
      }
      function installStyles() {
        if (document.getElementById("sns-video-v193-style")) {
          return;
        }
        var style = document.createElement("style");
        style.id = "sns-video-v193-style";
        style.textContent = [ "body.sns-chat-page #pun-viewtopic .sns-message.sns-video-message .post-body{", "width:fit-content!important;min-width:0!important;max-width:min(360px,92vw)!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-video-message .post-box{", "width:fit-content!important;min-width:0!important;max-width:min(360px,92vw)!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-video-message .post-content{", "width:fit-content!important;min-width:0!important;max-width:min(360px,92vw)!important;", "padding:5px!important;overflow:visible!important;", "}", "body.sns-chat-page .sns-video-card{", "display:block!important;width:fit-content!important;max-width:100%!important;", "box-sizing:border-box!important;color:inherit!important;", "}", "body.sns-chat-page .sns-video-native{", "display:block!important;width:auto!important;height:auto!important;", "max-width:340px!important;max-height:430px!important;", "margin:0!important;background:#0d0d0f!important;", "border:0!important;border-radius:12px!important;", "object-fit:contain!important;", "}", "body.sns-chat-page .sns-video-caption{", "display:block!important;margin:0!important;padding:8px 8px 5px!important;", "font:12px/1.4 Arial,sans-serif!important;", "white-space:pre-wrap!important;overflow-wrap:anywhere!important;", "color:inherit!important;", "}", "body.sns-chat-page .sns-video-caption:empty{display:none!important;}", "body.sns-chat-page .sns-video-card.is-video-error .sns-video-native{", "min-width:220px!important;min-height:120px!important;", "}", "body.sns-chat-page .sns-video-error-note{", "display:none!important;padding:7px 8px 5px!important;", "color:rgba(255,255,255,.76)!important;", "font:10px/1.35 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-video-card.is-video-error .sns-video-error-note{display:block!important;}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.sns-pending-video .sns-pending-bubble{", "padding:5px!important;", "}", "body.sns-chat-page #sns-pending-stack .sns-pending-video .sns-video-card{", "width:fit-content!important;max-width:min(340px,88vw)!important;", "}", "body.sns-chat-page #sns-video-compose{", "position:absolute!important;left:64px!important;right:58px!important;bottom:64px!important;", "z-index:2780!important;display:none!important;", "box-sizing:border-box!important;padding:11px!important;", "min-height:245px!important;max-height:calc(100vh - 150px)!important;", "overflow-y:auto!important;overflow-x:hidden!important;", "background:rgba(255,255,255,.985)!important;color:#333!important;", "border:1px solid rgba(0,0,0,.07)!important;border-radius:12px!important;", "box-shadow:0 10px 28px rgba(18,20,28,.12),0 22px 52px rgba(18,20,28,.13)!important;", "}", "body.sns-chat-page #sns-video-compose.is-open{display:block!important;}", "body.sns-chat-page .sns-video-compose-title{", "display:flex!important;align-items:center!important;gap:7px!important;", "margin:0 0 9px!important;color:#444!important;", "font:700 11px/1.2 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-video-compose-title svg{", "display:block!important;width:15px!important;height:15px!important;", "fill:none!important;stroke:var(--sns-own-g1,#b65f3a)!important;", "stroke-width:1.9!important;stroke-linecap:round!important;stroke-linejoin:round!important;", "}", "body.sns-chat-page .sns-video-upload-box{", "display:grid!important;grid-template-columns:auto minmax(0,1fr)!important;", "align-items:center!important;gap:9px!important;", "margin:0 0 9px!important;padding:8px 9px!important;", "background:#f6f6f7!important;border:1px solid rgba(0,0,0,.07)!important;", "border-radius:9px!important;", "}", "body.sns-chat-page .sns-video-file-input{display:none!important;}", "body.sns-chat-page .sns-video-file-pick{", "height:32px!important;margin:0!important;padding:0 11px!important;", "border:0!important;border-radius:8px!important;cursor:pointer!important;", "background:#2c2c31!important;color:#fff!important;", "font:700 9px/32px Arial,sans-serif!important;white-space:nowrap!important;", "}", "body.sns-chat-page .sns-video-file-pick:disabled{cursor:default!important;opacity:.38!important;}", "body.sns-chat-page .sns-video-upload-info{", "display:flex!important;flex-direction:column!important;gap:5px!important;min-width:0!important;", "}", "body.sns-chat-page .sns-video-upload-status{", "display:block!important;overflow:hidden!important;text-overflow:ellipsis!important;", "white-space:nowrap!important;color:#888!important;font:9px/1.2 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-video-upload-status.is-error{color:#b64e42!important;}", "body.sns-chat-page .sns-video-upload-status.is-success{color:#62815c!important;}", "body.sns-chat-page .sns-video-upload-progress{", "display:none!important;position:relative!important;width:100%!important;height:3px!important;", "overflow:hidden!important;background:rgba(0,0,0,.08)!important;border-radius:99px!important;", "}", "body.sns-chat-page .sns-video-upload-progress.is-visible{display:block!important;}", "body.sns-chat-page .sns-video-upload-progress>i{", "display:block!important;width:0;height:100%!important;", "background:var(--sns-own-g1,#b65f3a)!important;border-radius:inherit!important;", "transition:width .12s linear!important;", "}", "body.sns-chat-page .sns-video-fields{display:grid!important;grid-template-columns:1fr!important;gap:7px!important;}", "body.sns-chat-page .sns-video-field{", "display:flex!important;flex-direction:column!important;gap:4px!important;min-width:0!important;", "}", "body.sns-chat-page .sns-video-field label{color:#888!important;font:700 8px/1.2 Arial,sans-serif!important;}", "body.sns-chat-page .sns-video-field input,body.sns-chat-page .sns-video-field textarea{", "box-sizing:border-box!important;width:100%!important;margin:0!important;", "padding:0 9px!important;background:#f7f7f7!important;color:#333!important;", "border:1px solid rgba(0,0,0,.08)!important;border-radius:8px!important;", "outline:none!important;font:11px/1.35 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-video-field input{", "display:block!important;visibility:visible!important;opacity:1!important;", "height:36px!important;min-height:36px!important;line-height:36px!important;", "position:relative!important;z-index:1!important;", "}", "body.sns-chat-page .sns-video-caption-field{", "display:flex!important;visibility:visible!important;opacity:1!important;", "min-height:55px!important;", "}", "body.sns-chat-page #sns-video-compose .sns-video-caption-field{", "display:flex!important;visibility:visible!important;opacity:1!important;", "min-height:90px!important;height:auto!important;overflow:visible!important;", "}", "body.sns-chat-page #sns-video-compose textarea.sns-video-caption{", "display:block!important;visibility:visible!important;opacity:1!important;", "position:relative!important;z-index:2!important;", "width:100%!important;height:68px!important;min-height:68px!important;max-height:150px!important;", "box-sizing:border-box!important;margin:0!important;padding:9px 10px!important;", "background:#fff!important;color:#333!important;", "border:1px solid rgba(0,0,0,.16)!important;border-radius:8px!important;", "font:11px/1.35 Arial,sans-serif!important;", "line-height:1.35!important;resize:vertical!important;", "overflow:auto!important;", "}", "body.sns-chat-page .sns-video-field textarea{", "min-height:56px!important;max-height:115px!important;", "padding-top:8px!important;padding-bottom:8px!important;resize:vertical!important;", "}", "body.sns-chat-page .sns-video-field input:focus,body.sns-chat-page .sns-video-field textarea:focus{", "border-color:color-mix(in srgb,var(--sns-own-g1,#b65f3a) 42%,transparent)!important;", "}", "body.sns-chat-page .sns-video-field input.is-invalid{border-color:#c85b4a!important;}", "body.sns-chat-page .sns-video-compose-foot{", "display:flex!important;align-items:center!important;justify-content:space-between!important;", "gap:10px!important;margin-top:9px!important;", "}", "body.sns-chat-page .sns-video-compose-hint{", "min-width:0!important;color:#999!important;font:8px/1.3 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-video-compose-actions{", "display:flex!important;align-items:center!important;gap:5px!important;flex:0 0 auto!important;", "}", "body.sns-chat-page .sns-video-compose-actions button{", "height:29px!important;margin:0!important;padding:0 10px!important;", "border:0!important;border-radius:7px!important;cursor:pointer!important;", "font:700 9px/29px Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-video-cancel{background:transparent!important;color:#777!important;}", "body.sns-chat-page .sns-video-submit{background:var(--sns-own-g1,#b65f3a)!important;color:#fff!important;}", "body.sns-chat-page .sns-video-submit:disabled{opacity:.35!important;cursor:default!important;}", "@media(max-width:650px){", "body.sns-chat-page #pun-viewtopic .sns-message.sns-video-message .post-body{", "width:fit-content!important;min-width:0!important;max-width:88vw!important;", "}", "body.sns-chat-page .sns-video-card{width:fit-content!important;max-width:88vw!important;}", "body.sns-chat-page .sns-video-native{max-width:82vw!important;max-height:62vh!important;}", "body.sns-chat-page #sns-video-compose{left:8px!important;right:8px!important;bottom:61px!important;}", "body.sns-chat-page .sns-video-upload-box{grid-template-columns:1fr!important;gap:6px!important;}", "body.sns-chat-page .sns-video-file-pick{width:100%!important;}", "body.sns-chat-page .sns-video-compose-foot{align-items:flex-end!important;}", "body.sns-chat-page .sns-video-compose-hint{max-width:160px!important;}", "}" ].join("");
        document.head.appendChild(style);
      }
      function setStatus($status, $progress, textValue, state, percent) {
        $status.removeClass("is-error is-success").toggleClass("is-error", state === "error").toggleClass("is-success", state === "success").text(textValue);
        var uploading = state === "uploading";
        $progress.toggleClass("is-visible", uploading);
        $progress.find("i").css("width", Math.max(0, Math.min(100, Number(percent) || 0)) + "%");
      }
      function uploadFile(file, $status, $progress) {
        var deferred = $.Deferred();
        var cfg = videoConfig();
        if (!cfg.cloudName || !cfg.uploadPreset) {
          deferred.reject("Cloudinary \u0434\u043b\u044f \u0432\u0438\u0434\u0435\u043e \u043d\u0435 \u043d\u0430\u0441\u0442\u0440\u043e\u0435\u043d");
          return deferred.promise();
        }
        if (!fileAllowed(file)) {
          deferred.reject("\u041d\u0443\u0436\u0435\u043d \u0432\u0438\u0434\u0435\u043e\u0444\u0430\u0439\u043b: MP4 / WebM / MOV / M4V / OGV");
          return deferred.promise();
        }
        if (Number(file.size) > cfg.maxMb * 1024 * 1024) {
          deferred.reject("\u0424\u0430\u0439\u043b \u0431\u043e\u043b\u044c\u0448\u0435 " + cfg.maxMb + " MB");
          return deferred.promise();
        }
        var endpoint = "https://api.cloudinary.com/v1_1/" + encodeURIComponent(cfg.cloudName) + "/video/upload";
        var formData = new FormData;
        formData.append("file", file);
        formData.append("upload_preset", cfg.uploadPreset);
        var xhr = new XMLHttpRequest;
        uploadXhr = xhr;
        uploadBusy = true;
        setStatus($status, $progress, "\u0417\u0430\u0433\u0440\u0443\u0436\u0430\u044e " + String(file.name || "\u0432\u0438\u0434\u0435\u043e") + "\u2026", "uploading", 0);
        xhr.open("POST", endpoint, true);
        xhr.upload.onprogress = function(event) {
          if (!event.lengthComputable) {
            return;
          }
          var percent = event.loaded / event.total * 100;
          setStatus($status, $progress, "\u0417\u0430\u0433\u0440\u0443\u0436\u0430\u044e " + String(file.name || "\u0432\u0438\u0434\u0435\u043e") + "\u2026 " + Math.round(percent) + "%", "uploading", percent);
        };
        xhr.onload = function() {
          uploadBusy = false;
          uploadXhr = null;
          var response = null;
          try {
            response = JSON.parse(xhr.responseText || "{}");
          } catch (error) {}
          if (xhr.status >= 200 && xhr.status < 300 && response && response.secure_url) {
            deferred.resolve(response);
          } else {
            deferred.reject(response && response.error && response.error.message ? response.error.message : "Cloudinary \u0432\u0435\u0440\u043d\u0443\u043b \u043e\u0448\u0438\u0431\u043a\u0443 " + xhr.status);
          }
        };
        xhr.onerror = function() {
          uploadBusy = false;
          uploadXhr = null;
          deferred.reject("\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u0432\u0438\u0434\u0435\u043e");
        };
        xhr.onabort = function() {
          uploadBusy = false;
          uploadXhr = null;
          deferred.reject("\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u043e\u0442\u043c\u0435\u043d\u0435\u043d\u0430");
        };
        xhr.send(formData);
        return deferred.promise();
      }
      function installUi() {
        if (uiInstalled) {
          return true;
        }
        var $shell = $("#sns-chat-shell").first();
        var $menu = $("#sns-attach-menu").first();
        var $input = $("#sns-ui-input").first();
        var $send = $(".sns-ui-send").first();
        if (!$shell.length || !$menu.length || !$input.length || !$send.length) {
          return false;
        }
        installStyles();
        if (!$menu.find(".sns-attach-video").length) {
          var $button = $('<button type="button" class="sns-attach-video">' + '<span class="sns-attach-icon" aria-hidden="true">' + '<svg viewBox="0 0 24 24">' + '<rect x="3.5" y="5.5" width="13" height="13" rx="2"></rect>' + '<path d="m16.5 10 4-2v8l-4-2z"></path>' + "</svg>" + "</span>" + '<span class="sns-attach-label">\u0412\u0438\u0434\u0435\u043e</span>' + "</button>");
          var $photo = $menu.find(".sns-attach-photo").first();
          if ($photo.length) {
            $button.insertAfter($photo);
          } else {
            $menu.prepend($button);
          }
        }
        var $panel = $("#sns-video-compose");
        if (!$panel.length) {
          $panel = $('<div id="sns-video-compose" aria-hidden="true">' + '<div class="sns-video-compose-title">' + '<svg viewBox="0 0 24 24" aria-hidden="true">' + '<rect x="3.5" y="5.5" width="13" height="13" rx="2"></rect>' + '<path d="m16.5 10 4-2v8l-4-2z"></path>' + "</svg>" + "<span>\u0412\u0438\u0434\u0435\u043e</span>" + "</div>" + '<div class="sns-video-upload-box">' + '<input class="sns-video-file-input" type="file" accept=".mp4,.webm,.mov,.m4v,.ogv,video/*">' + '<button type="button" class="sns-video-file-pick">\u0417\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u0441 \u043a\u043e\u043c\u043f\u044c\u044e\u0442\u0435\u0440\u0430</button>' + '<div class="sns-video-upload-info">' + '<span class="sns-video-upload-status">\u0418\u043b\u0438 \u0432\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u043f\u0440\u044f\u043c\u0443\u044e \u0441\u0441\u044b\u043b\u043a\u0443 \u043d\u0438\u0436\u0435</span>' + '<span class="sns-video-upload-progress"><i></i></span>' + "</div>" + "</div>" + '<div class="sns-video-fields">' + '<div class="sns-video-field">' + "<label>\u041f\u0440\u044f\u043c\u0430\u044f \u0441\u0441\u044b\u043b\u043a\u0430 \u043d\u0430 \u0432\u0438\u0434\u0435\u043e *</label>" + '<input class="sns-video-url" type="url" placeholder="https://site.com/video.mp4">' + "</div>" + '<div class="sns-video-field sns-video-caption-field">' + "<label>\u041f\u043e\u0434\u043f\u0438\u0441\u044c \u043a \u0432\u0438\u0434\u0435\u043e (\u043d\u0435\u043e\u0431\u044f\u0437\u0430\u0442\u0435\u043b\u044c\u043d\u043e)</label>" + '<textarea class="sns-video-caption" maxlength="3000" rows="3" ' + 'placeholder="\u041d\u0430\u043f\u0438\u0448\u0438\u0442\u0435 \u043f\u043e\u0434\u043f\u0438\u0441\u044c \u043a \u0432\u0438\u0434\u0435\u043e..." ' + 'style="display:block!important;visibility:visible!important;opacity:1!important;' + "width:100%!important;height:68px!important;min-height:68px!important;" + "box-sizing:border-box!important;margin:0!important;padding:9px 10px!important;" + "background:#ffffff!important;color:#333333!important;" + "border:1px solid rgba(0,0,0,.16)!important;border-radius:8px!important;" + 'font:11px/1.35 Arial,sans-serif!important;resize:vertical!important;"></textarea>' + "</div>" + "</div>" + '<div class="sns-video-compose-foot">' + '<span class="sns-video-compose-hint"></span>' + '<span class="sns-video-compose-actions">' + '<button type="button" class="sns-video-cancel">\u041e\u0442\u043c\u0435\u043d\u0430</button>' + '<button type="button" class="sns-video-submit" disabled>\u041e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c</button>' + "</span>" + "</div>" + "</div>");
          $shell.append($panel);
        }
        var $fileInput = $panel.find(".sns-video-file-input");
        var $filePick = $panel.find(".sns-video-file-pick");
        var $status = $panel.find(".sns-video-upload-status");
        var $progress = $panel.find(".sns-video-upload-progress");
        var $url = $panel.find(".sns-video-url");
        var $caption = $panel.find(".sns-video-caption");
        var $submit = $panel.find(".sns-video-submit");
        var $hint = $panel.find(".sns-video-compose-hint");
        function refresh() {
          var cfg = videoConfig();
          var ready = !!(cfg.cloudName && cfg.uploadPreset);
          $filePick.prop("disabled", !ready || uploadBusy).attr("title", ready ? "\u041c\u0430\u043a\u0441. " + cfg.maxMb + " MB" : "\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u0441 \u043a\u043e\u043c\u043f\u044c\u044e\u0442\u0435\u0440\u0430 \u043d\u0435 \u043d\u0430\u0441\u0442\u0440\u043e\u0435\u043d\u0430");
          $hint.text(ready ? "MP4 / WebM / MOV. \u041c\u0430\u043a\u0441\u0438\u043c\u0443\u043c " + cfg.maxMb + " MB." : "\u041c\u043e\u0436\u043d\u043e \u043e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c \u0432\u0438\u0434\u0435\u043e \u043f\u043e \u043f\u0440\u044f\u043c\u043e\u0439 https-\u0441\u0441\u044b\u043b\u043a\u0435.");
          var valid = !!safeUrl($url.val());
          $submit.prop("disabled", !valid || uploadBusy);
          $url.toggleClass("is-invalid", !!$.trim($url.val()) && !valid);
        }
        function closePanel(clear) {
          $panel.removeClass("is-open").attr("aria-hidden", "true");
          if (clear) {
            if (uploadXhr && uploadBusy) {
              try {
                uploadXhr.abort();
              } catch (error) {}
            }
            uploadXhr = null;
            uploadBusy = false;
            $fileInput.val("");
            $url.val("");
            $caption.val("");
            setStatus($status, $progress, cloudinaryReady() ? "\u0418\u043b\u0438 \u0432\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u043f\u0440\u044f\u043c\u0443\u044e \u0441\u0441\u044b\u043b\u043a\u0443 \u043d\u0438\u0436\u0435" : "\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u0441 \u043a\u043e\u043c\u043f\u044c\u044e\u0442\u0435\u0440\u0430 \u043d\u0435 \u043d\u0430\u0441\u0442\u0440\u043e\u0435\u043d\u0430; \u0441\u0441\u044b\u043b\u043a\u0443 \u043c\u043e\u0436\u043d\u043e \u0432\u0441\u0442\u0430\u0432\u0438\u0442\u044c \u0432\u0440\u0443\u0447\u043d\u0443\u044e", "", 0);
          }
          refresh();
        }
        function openPanel() {
          $("#sns-format-toolbar").removeClass("is-open");
          $("#sns-voice-compose,#sns-audio-compose,#sns-story-compose").removeClass("is-open").attr("aria-hidden", "true");
          $menu.removeClass("is-open");
          $(".sns-ui-plus").removeClass("is-active");
          $panel.addClass("is-open").attr("aria-hidden", "false");
          refresh();
          setTimeout(function() {
            if ($url.val()) {
              $caption.focus();
            } else {
              $url.focus();
            }
          }, 20);
        }
        $menu.off("click.snsVideoV193", ".sns-attach-video").on("click.snsVideoV193", ".sns-attach-video", function(event) {
          event.preventDefault();
          event.stopPropagation();
          openPanel();
        });
        $filePick.off(".snsVideoV193").on("click.snsVideoV193", function(event) {
          event.preventDefault();
          event.stopPropagation();
          if (!cloudinaryReady() || uploadBusy) {
            return;
          }
          $fileInput.trigger("click");
        });
        $fileInput.off(".snsVideoV193").on("change.snsVideoV193", function() {
          var file = this.files && this.files[0] ? this.files[0] : null;
          if (!file) {
            return;
          }
          refresh();
          uploadFile(file, $status, $progress).done(function(response) {
            var url = deliveryUrl(response);
            $url.val(url).removeClass("is-invalid");
            setStatus($status, $progress, "\u0413\u043e\u0442\u043e\u0432\u043e \u2014 \u0442\u0435\u043f\u0435\u0440\u044c \u043c\u043e\u0436\u043d\u043e \u0434\u043e\u0431\u0430\u0432\u0438\u0442\u044c \u043f\u043e\u0434\u043f\u0438\u0441\u044c \u043d\u0438\u0436\u0435", "success", 100);
            refresh();
            setTimeout(function() {
              $caption.focus();
            }, 40);
          }).fail(function(message) {
            if (String(message || "") === "\u0417\u0430\u0433\u0440\u0443\u0437\u043a\u0430 \u043e\u0442\u043c\u0435\u043d\u0435\u043d\u0430") {
              return;
            }
            setStatus($status, $progress, String(message || "\u041e\u0448\u0438\u0431\u043a\u0430 \u0437\u0430\u0433\u0440\u0443\u0437\u043a\u0438"), "error", 0);
            refresh();
          });
        });
        $panel.off(".snsVideoV193").on("input.snsVideoV193", "input,textarea", refresh).on("click.snsVideoV193", ".sns-video-cancel", function(event) {
          event.preventDefault();
          event.stopPropagation();
          closePanel(true);
          $input.focus();
        }).on("click.snsVideoV193", ".sns-video-submit", function(event) {
          event.preventDefault();
          event.stopPropagation();
          if (uploadBusy) {
            return;
          }
          var marker = markerFromData({
            url: $url.val(),
            caption: $caption.val()
          });
          if (!marker) {
            $url.addClass("is-invalid").focus();
            return;
          }
          closePanel(true);
          $input.val(marker).trigger("input");
          setTimeout(function() {
            $send.trigger("click");
          }, 0);
        }).on("keydown.snsVideoV193", "input,textarea", function(event) {
          if (event.key === "Escape") {
            event.preventDefault();
            closePanel(false);
            $input.focus();
          }
          if ((event.ctrlKey || event.metaKey) && event.key === "Enter") {
            event.preventDefault();
            $submit.trigger("click");
          }
        });
        $(document).off("mousedown.snsVideoCloseV193").on("mousedown.snsVideoCloseV193", function(event) {
          if ($panel.hasClass("is-open") && !$(event.target).closest("#sns-video-compose,.sns-attach-video").length) {
            closePanel(false);
          }
        });
        $(document).off("click.snsVideoOtherPanelsV193").on("click.snsVideoOtherPanelsV193", ".sns-ui-plus,.sns-ui-format,.sns-ui-storytime,.sns-attach-photo,.sns-attach-voice,.sns-attach-audio", function() {
          if (!$(this).hasClass("sns-attach-video")) {
            closePanel(false);
          }
        });
        refresh();
        uiInstalled = true;
        return true;
      }
      function installObserver() {
        if (observer || !window.MutationObserver) {
          return;
        }
        var root = document.getElementById("sns-chat-shell");
        if (!root) {
          return;
        }
        observer = new MutationObserver(function(mutations) {
          var relevant = false;
          for (var i = 0; i < mutations.length; i++) {
            if (mutations[i].addedNodes && mutations[i].addedNodes.length) {
              relevant = true;
              break;
            }
          }
          if (relevant) {
            scheduleRender(35);
          }
        });
        observer.observe(root, {
          childList: true,
          subtree: true
        });
      }
      function start() {
        if (!document.body.classList.contains("sns-chat-page")) {
          return;
        }
        installStyles();
        if (!installUi()) {
          return;
        }
        renderAll();
        $(document).off("sns_pages_loaded.snsVideoV193 pun_edit.snsVideoV193").on("sns_pages_loaded.snsVideoV193 pun_edit.snsVideoV193", function() {
          scheduleRender(70);
        });
        setTimeout(renderAll, 300);
      }
      window.SNSVideoEncodeMarker = markerFromData;
      window.SNSVideoDecodeMarker = function(value) {
        var parsed = parseMarker(value);
        return parsed ? parsed.data : null;
      };
      start();
    })(jQuery);
    (function($) {
      "use strict";
      var LOCATION_PREFIX = "SNSLOC:";
      var renderTimer = null;
      var observer = null;
      var uiInstalled = false;
      function encodeText(value) {
        try {
          return btoa(unescape(encodeURIComponent(String(value || ""))));
        } catch (error) {
          return "";
        }
      }
      function decodeText(value) {
        try {
          return decodeURIComponent(escape(atob(String(value || ""))));
        } catch (error) {
          return null;
        }
      }
      function clean(value) {
        return String(value || "").replace(/\s+/g, " ").trim();
      }
      function normalizeLocationData(data) {
        data = data && typeof data === "object" ? data : {};
        var place = clean(data.place !== undefined ? data.place : data.p).slice(0, 180);
        var detail = clean(data.detail !== undefined ? data.detail : data.d).slice(0, 300);
        if (!place) {
          return null;
        }
        return {
          place: place,
          detail: detail
        };
      }
      function markerFromData(data) {
        var normalized = normalizeLocationData(data);
        if (!normalized) {
          return "";
        }
        var encoded = encodeText(JSON.stringify({
          p: normalized.place,
          d: normalized.detail
        }));
        return encoded ? LOCATION_PREFIX + encoded : "";
      }
      function parseMarker(value) {
        var raw = String(value || "");
        var match = raw.match(/SNSLOC:([A-Za-z0-9+\/=]+)/);
        if (!match) {
          return null;
        }
        var decoded = decodeText(match[1]);
        if (decoded === null) {
          return null;
        }
        try {
          var data = normalizeLocationData(JSON.parse(decoded));
          return data ? {
            encoded: match[1],
            data: data
          } : null;
        } catch (error) {
          return null;
        }
      }
      function createLocationCard(data) {
        var normalized = normalizeLocationData(data);
        if (!normalized) {
          return $("<div></div>");
        }
        var $card = $('<div class="sns-location-card">' + '<div class="sns-location-map" aria-hidden="true">' + '<span class="sns-location-road r1"></span>' + '<span class="sns-location-road r2"></span>' + '<span class="sns-location-road r3"></span>' + '<span class="sns-location-road r4"></span>' + '<span class="sns-location-block b1"></span>' + '<span class="sns-location-block b2"></span>' + '<span class="sns-location-block b3"></span>' + '<span class="sns-location-pin">' + '<svg viewBox="0 0 24 24">' + '<path d="M12 21s6-5.35 6-11a6 6 0 1 0-12 0c0 5.65 6 11 6 11z"></path>' + '<circle cx="12" cy="10" r="2.2"></circle>' + "</svg>" + "</span>" + "</div>" + '<div class="sns-location-info">' + '<div class="sns-location-kicker">\u041b\u041e\u041a\u0410\u0426\u0418\u042f</div>' + '<div class="sns-location-place"></div>' + '<div class="sns-location-detail"></div>' + "</div>" + "</div>");
        $card.find(".sns-location-place").text(normalized.place);
        $card.find(".sns-location-detail").text(normalized.detail);
        return $card;
      }
      function renderRealLocations() {
        $("#pun-viewtopic .post.sns-message").each(function() {
          var $post = $(this);
          var $content = $post.find(".post-content").first();
          if (!$content.length) {
            return;
          }
          var $existing = $content.children(".sns-location-card").first();
          if ($existing.length) {
            return;
          }
          var payload = parseMarker($content.text());
          if (!payload || !payload.data) {
            return;
          }
          $post.addClass("sns-location-message").removeClass("sns-has-media sns-media-only sns-media-caption sns-image-only").attr("data-sns-location", payload.encoded).attr("data-sns-images", "0");
          $content.empty().append(createLocationCard(payload.data));
        });
      }
      function renderPendingLocations() {
        $("#sns-pending-stack .sns-pending-row").each(function() {
          var $row = $(this);
          var $bubble = $row.find(".sns-pending-bubble").first();
          if (!$bubble.length) {
            return;
          }
          if ($bubble.children(".sns-location-card").length) {
            return;
          }
          var payload = parseMarker($bubble.text());
          if (!payload || !payload.data) {
            return;
          }
          $row.addClass("sns-pending-location");
          $bubble.empty().append(createLocationCard(payload.data));
        });
      }
      function renderAll() {
        renderRealLocations();
        renderPendingLocations();
      }
      function scheduleRender(delay) {
        if (renderTimer) {
          clearTimeout(renderTimer);
        }
        renderTimer = setTimeout(function() {
          renderTimer = null;
          renderAll();
        }, delay === undefined ? 45 : delay);
      }
      function installStyles() {
        if (document.getElementById("sns-location-v198-style")) {
          return;
        }
        var style = document.createElement("style");
        style.id = "sns-location-v198-style";
        style.textContent = [ "body.sns-chat-page #pun-viewtopic .sns-message.sns-location-message .post-body{", "width:300px!important;min-width:300px!important;max-width:300px!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-location-message .post-content{", "padding:5px!important;overflow:visible!important;", "}", "body.sns-chat-page .sns-location-card{", "display:block!important;box-sizing:border-box!important;", "width:290px!important;max-width:100%!important;", "overflow:hidden!important;", "background:rgba(255,255,255,.12)!important;color:inherit!important;", "border-radius:13px!important;", "}", "body.sns-chat-page .sns-location-map{", "position:relative!important;display:block!important;", "width:100%!important;height:132px!important;overflow:hidden!important;", "background:", "linear-gradient(135deg,rgba(255,255,255,.045) 25%,transparent 25%) 0 0/28px 28px,", "linear-gradient(315deg,rgba(255,255,255,.035) 25%,transparent 25%) 0 0/34px 34px,", "linear-gradient(180deg,rgba(30,34,38,.96),rgba(43,46,50,.94))!important;", "}", "body.sns-chat-page .sns-other .sns-location-map{", "background:", "linear-gradient(135deg,rgba(0,0,0,.035) 25%,transparent 25%) 0 0/28px 28px,", "linear-gradient(315deg,rgba(0,0,0,.026) 25%,transparent 25%) 0 0/34px 34px,", "linear-gradient(180deg,#e7e5df,#dcdad3)!important;", "}", "body.sns-chat-page .sns-location-road{", "position:absolute!important;display:block!important;", "height:5px!important;background:rgba(255,255,255,.16)!important;", "border-radius:99px!important;transform-origin:center!important;", "}", "body.sns-chat-page .sns-other .sns-location-road{background:rgba(87,84,78,.17)!important;}", "body.sns-chat-page .sns-location-road.r1{width:250px!important;left:-28px!important;top:34px!important;transform:rotate(15deg)!important;}", "body.sns-chat-page .sns-location-road.r2{width:220px!important;left:64px!important;top:91px!important;transform:rotate(-18deg)!important;}", "body.sns-chat-page .sns-location-road.r3{width:170px!important;left:-24px!important;top:101px!important;transform:rotate(-33deg)!important;}", "body.sns-chat-page .sns-location-road.r4{width:150px!important;right:-34px!important;top:43px!important;transform:rotate(53deg)!important;}", "body.sns-chat-page .sns-location-block{", "position:absolute!important;display:block!important;", "border:1px solid rgba(255,255,255,.11)!important;", "border-radius:5px!important;background:rgba(255,255,255,.045)!important;", "}", "body.sns-chat-page .sns-other .sns-location-block{", "border-color:rgba(80,76,70,.12)!important;background:rgba(255,255,255,.24)!important;", "}", "body.sns-chat-page .sns-location-block.b1{width:53px!important;height:31px!important;left:25px!important;top:53px!important;transform:rotate(8deg)!important;}", "body.sns-chat-page .sns-location-block.b2{width:66px!important;height:36px!important;right:29px!important;top:17px!important;transform:rotate(-7deg)!important;}", "body.sns-chat-page .sns-location-block.b3{width:49px!important;height:34px!important;right:55px!important;bottom:14px!important;transform:rotate(11deg)!important;}", "body.sns-chat-page .sns-location-pin{", "position:absolute!important;left:50%!important;top:50%!important;", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:39px!important;height:39px!important;", "transform:translate(-50%,-55%)!important;", "border-radius:50%!important;", "background:var(--sns-own-g1,#b65f3a)!important;color:#fff!important;", "box-shadow:0 6px 16px rgba(0,0,0,.25)!important;", "}", "body.sns-chat-page .sns-location-pin svg{", "display:block!important;width:22px!important;height:22px!important;", "fill:none!important;stroke:currentColor!important;stroke-width:1.8!important;", "stroke-linecap:round!important;stroke-linejoin:round!important;", "}", "body.sns-chat-page .sns-location-info{", "display:block!important;box-sizing:border-box!important;", "padding:10px 11px 11px!important;", "}", "body.sns-chat-page .sns-location-kicker{", "margin:0 0 3px!important;opacity:.54!important;", "font:700 7px/1.2 Arial,sans-serif!important;letter-spacing:1.1px!important;", "}", "body.sns-chat-page .sns-location-place{", "margin:0!important;font:700 12px/1.3 Arial,sans-serif!important;", "overflow-wrap:anywhere!important;", "}", "body.sns-chat-page .sns-location-detail{", "margin:4px 0 0!important;opacity:.7!important;", "font:9px/1.35 Arial,sans-serif!important;overflow-wrap:anywhere!important;", "}", "body.sns-chat-page .sns-location-detail:empty{display:none!important;}", "body.sns-chat-page #sns-pending-stack .sns-pending-row.sns-pending-location .sns-pending-bubble{", "padding:5px!important;width:300px!important;max-width:88vw!important;", "}", "body.sns-chat-page #sns-location-compose{", "position:absolute!important;left:64px!important;right:58px!important;bottom:64px!important;", "z-index:2790!important;display:none!important;", "box-sizing:border-box!important;padding:11px!important;", "background:rgba(255,255,255,.985)!important;color:#333!important;", "border:1px solid rgba(0,0,0,.07)!important;border-radius:12px!important;", "box-shadow:0 10px 28px rgba(18,20,28,.12),0 22px 52px rgba(18,20,28,.13)!important;", "}", "body.sns-chat-page #sns-location-compose.is-open{display:block!important;}", "body.sns-chat-page .sns-location-compose-title{", "display:flex!important;align-items:center!important;gap:7px!important;", "margin:0 0 9px!important;color:#444!important;", "font:700 11px/1.2 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-location-compose-title svg{", "display:block!important;width:15px!important;height:15px!important;", "fill:none!important;stroke:var(--sns-own-g1,#b65f3a)!important;", "stroke-width:1.9!important;stroke-linecap:round!important;stroke-linejoin:round!important;", "}", "body.sns-chat-page .sns-location-compose-note{", "margin:-2px 0 9px!important;color:#999!important;", "font:8px/1.35 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-location-fields{display:grid!important;grid-template-columns:1fr!important;gap:7px!important;}", "body.sns-chat-page .sns-location-field{", "display:flex!important;flex-direction:column!important;gap:4px!important;min-width:0!important;", "}", "body.sns-chat-page .sns-location-field label{color:#888!important;font:700 8px/1.2 Arial,sans-serif!important;}", "body.sns-chat-page #sns-location-compose .sns-location-field input{", "display:block!important;visibility:visible!important;opacity:1!important;", "box-sizing:border-box!important;width:100%!important;height:37px!important;min-height:37px!important;", "margin:0!important;padding:0 10px!important;", "background:#fff!important;color:#333!important;", "border:1px solid rgba(0,0,0,.12)!important;border-radius:8px!important;", "outline:none!important;font:11px/37px Arial,sans-serif!important;", "}", "body.sns-chat-page #sns-location-compose .sns-location-field input:focus{", "border-color:color-mix(in srgb,var(--sns-own-g1,#b65f3a) 42%,transparent)!important;", "}", "body.sns-chat-page #sns-location-compose .sns-location-place-input.is-invalid{", "border-color:#c85b4a!important;", "}", "body.sns-chat-page .sns-location-compose-foot{", "display:flex!important;align-items:center!important;justify-content:space-between!important;", "gap:10px!important;margin-top:9px!important;", "}", "body.sns-chat-page .sns-location-compose-hint{", "min-width:0!important;color:#999!important;font:8px/1.3 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-location-compose-actions{", "display:flex!important;align-items:center!important;gap:5px!important;flex:0 0 auto!important;", "}", "body.sns-chat-page .sns-location-compose-actions button{", "height:29px!important;margin:0!important;padding:0 10px!important;", "border:0!important;border-radius:7px!important;cursor:pointer!important;", "font:700 9px/29px Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-location-cancel{background:transparent!important;color:#777!important;}", "body.sns-chat-page .sns-location-submit{background:var(--sns-own-g1,#b65f3a)!important;color:#fff!important;}", "body.sns-chat-page .sns-location-submit:disabled{opacity:.35!important;cursor:default!important;}", "@media(max-width:650px){", "body.sns-chat-page #pun-viewtopic .sns-message.sns-location-message .post-body{", "width:270px!important;min-width:270px!important;max-width:270px!important;", "}", "body.sns-chat-page .sns-location-card{width:260px!important;}", "body.sns-chat-page .sns-location-map{height:118px!important;}", "body.sns-chat-page #sns-location-compose{left:8px!important;right:8px!important;bottom:61px!important;}", "}" ].join("");
        document.head.appendChild(style);
      }
      function installUi() {
        if (uiInstalled) {
          return true;
        }
        var $shell = $("#sns-chat-shell").first();
        var $menu = $("#sns-attach-menu").first();
        var $input = $("#sns-ui-input").first();
        var $send = $(".sns-ui-send").first();
        if (!$shell.length || !$menu.length || !$input.length || !$send.length) {
          return false;
        }
        installStyles();
        if (!$menu.find(".sns-attach-location").length) {
          var $button = $('<button type="button" class="sns-attach-location">' + '<span class="sns-attach-icon" aria-hidden="true">' + '<svg viewBox="0 0 24 24">' + '<path d="M12 21s6-5.35 6-11a6 6 0 1 0-12 0c0 5.65 6 11 6 11z"></path>' + '<circle cx="12" cy="10" r="2.2"></circle>' + "</svg>" + "</span>" + '<span class="sns-attach-label">\u041b\u043e\u043a\u0430\u0446\u0438\u044f</span>' + "</button>");
          $menu.append($button);
        }
        var $panel = $("#sns-location-compose");
        if (!$panel.length) {
          $panel = $('<div id="sns-location-compose" aria-hidden="true">' + '<div class="sns-location-compose-title">' + '<svg viewBox="0 0 24 24" aria-hidden="true">' + '<path d="M12 21s6-5.35 6-11a6 6 0 1 0-12 0c0 5.65 6 11 6 11z"></path>' + '<circle cx="12" cy="10" r="2.2"></circle>' + "</svg>" + "<span>\u0418\u0433\u0440\u043e\u0432\u0430\u044f \u043b\u043e\u043a\u0430\u0446\u0438\u044f</span>" + "</div>" + '<div class="sns-location-compose-note">' + "\u042d\u0442\u043e \u0441\u044e\u0436\u0435\u0442\u043d\u0430\u044f \u043c\u0435\u0442\u043a\u0430 \u2014 \u0440\u0435\u0430\u043b\u044c\u043d\u0430\u044f \u0433\u0435\u043e\u043b\u043e\u043a\u0430\u0446\u0438\u044f \u0443\u0441\u0442\u0440\u043e\u0439\u0441\u0442\u0432\u0430 \u043d\u0435 \u0438\u0441\u043f\u043e\u043b\u044c\u0437\u0443\u0435\u0442\u0441\u044f." + "</div>" + '<div class="sns-location-fields">' + '<div class="sns-location-field">' + "<label>\u041c\u0435\u0441\u0442\u043e *</label>" + '<input class="sns-location-place-input" type="text" maxlength="180" placeholder="Central Park, NYC">' + "</div>" + '<div class="sns-location-field">' + "<label>\u0410\u0434\u0440\u0435\u0441 / \u0443\u0442\u043e\u0447\u043d\u0435\u043d\u0438\u0435 (\u043d\u0435\u043e\u0431\u044f\u0437\u0430\u0442\u0435\u043b\u044c\u043d\u043e)</label>" + '<input class="sns-location-detail-input" type="text" maxlength="300" placeholder="Bethesda Terrace \xb7 Manhattan">' + "</div>" + "</div>" + '<div class="sns-location-compose-foot">' + '<span class="sns-location-compose-hint">\u0412 \u0447\u0430\u0442 \u0443\u0439\u0434\u0435\u0442 \u043a\u0430\u0440\u0442\u043e\u0447\u043a\u0430 \u043c\u0435\u0441\u0442\u0430, \u0430 \u043d\u0435 \u043d\u0430\u0441\u0442\u043e\u044f\u0449\u0438\u0439 GPS.</span>' + '<span class="sns-location-compose-actions">' + '<button type="button" class="sns-location-cancel">\u041e\u0442\u043c\u0435\u043d\u0430</button>' + '<button type="button" class="sns-location-submit" disabled>\u041e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c</button>' + "</span>" + "</div>" + "</div>");
          $shell.append($panel);
        }
        var $place = $panel.find(".sns-location-place-input");
        var $detail = $panel.find(".sns-location-detail-input");
        var $submit = $panel.find(".sns-location-submit");
        function refresh() {
          var valid = !!clean($place.val());
          $submit.prop("disabled", !valid);
          $place.toggleClass("is-invalid", !!$place.val() && !valid);
        }
        function closePanel(clear) {
          $panel.removeClass("is-open").attr("aria-hidden", "true");
          if (clear) {
            $place.val("");
            $detail.val("");
          }
          refresh();
        }
        function openPanel() {
          $("#sns-format-toolbar").removeClass("is-open");
          $("#sns-video-compose,#sns-voice-compose,#sns-audio-compose,#sns-story-compose").removeClass("is-open").attr("aria-hidden", "true");
          $menu.removeClass("is-open");
          $(".sns-ui-plus").removeClass("is-active");
          $panel.addClass("is-open").attr("aria-hidden", "false");
          refresh();
          setTimeout(function() {
            $place.focus();
          }, 20);
        }
        $menu.off("click.snsLocationV198", ".sns-attach-location").on("click.snsLocationV198", ".sns-attach-location", function(event) {
          event.preventDefault();
          event.stopPropagation();
          openPanel();
        });
        $panel.off(".snsLocationV198").on("input.snsLocationV198", "input", refresh).on("click.snsLocationV198", ".sns-location-cancel", function(event) {
          event.preventDefault();
          event.stopPropagation();
          closePanel(true);
          $input.focus();
        }).on("click.snsLocationV198", ".sns-location-submit", function(event) {
          event.preventDefault();
          event.stopPropagation();
          var marker = markerFromData({
            place: $place.val(),
            detail: $detail.val()
          });
          if (!marker) {
            $place.addClass("is-invalid").focus();
            return;
          }
          closePanel(true);
          $input.val(marker).trigger("input");
          setTimeout(function() {
            $send.trigger("click");
          }, 0);
        }).on("keydown.snsLocationV198", "input", function(event) {
          if (event.key === "Escape") {
            event.preventDefault();
            closePanel(false);
            $input.focus();
          }
          if (event.key === "Enter" && !event.shiftKey) {
            event.preventDefault();
            $submit.trigger("click");
          }
        });
        $(document).off("mousedown.snsLocationCloseV198").on("mousedown.snsLocationCloseV198", function(event) {
          if ($panel.hasClass("is-open") && !$(event.target).closest("#sns-location-compose,.sns-attach-location").length) {
            closePanel(false);
          }
        });
        $(document).off("click.snsLocationOtherPanelsV198").on("click.snsLocationOtherPanelsV198", ".sns-ui-plus,.sns-ui-format,.sns-ui-storytime,.sns-attach-photo,.sns-attach-video,.sns-attach-voice,.sns-attach-audio", function() {
          if (!$(this).hasClass("sns-attach-location")) {
            closePanel(false);
          }
        });
        refresh();
        uiInstalled = true;
        return true;
      }
      function installObserver() {
        if (observer || !window.MutationObserver) {
          return;
        }
        var root = document.getElementById("sns-chat-shell");
        if (!root) {
          return;
        }
        observer = new MutationObserver(function(mutations) {
          var relevant = false;
          for (var i = 0; i < mutations.length; i++) {
            if (mutations[i].addedNodes && mutations[i].addedNodes.length) {
              relevant = true;
              break;
            }
          }
          if (relevant) {
            scheduleRender(35);
          }
        });
        observer.observe(root, {
          childList: true,
          subtree: true
        });
      }
      function start() {
        if (!document.body.classList.contains("sns-chat-page")) {
          return;
        }
        installStyles();
        if (!installUi()) {
          return;
        }
        renderAll();
        $(document).off("sns_pages_loaded.snsLocationV198 pun_edit.snsLocationV198").on("sns_pages_loaded.snsLocationV198 pun_edit.snsLocationV198", function() {
          scheduleRender(70);
        });
        setTimeout(renderAll, 300);
      }
      window.SNSLocationEncodeMarker = markerFromData;
      window.SNSLocationDecodeMarker = function(value) {
        var parsed = parseMarker(value);
        return parsed ? parsed.data : null;
      };
      start();
    })(jQuery);
    (function() {
      "use strict";
      if (document.getElementById("sns-lazy-history-v205-style")) {
        return;
      }
      var style = document.createElement("style");
      style.id = "sns-lazy-history-v205-style";
      style.textContent = [ "body.sns-chat-page #sns-history-loader-v205{", "position:relative!important;", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:max-content!important;max-width:calc(100% - 28px)!important;", "min-height:25px!important;", "box-sizing:border-box!important;", "margin:10px auto 14px!important;padding:5px 11px!important;", "border:1px solid rgba(255,255,255,.10)!important;", "border-radius:999px!important;", "background:rgba(255,255,255,.075)!important;", "color:rgba(255,255,255,.62)!important;", "box-shadow:none!important;", "font:700 8px/1.2 Arial,sans-serif!important;", "letter-spacing:.25px!important;", "cursor:pointer!important;z-index:4!important;", "}", "body.sns-chat-page #sns-history-loader-v205:hover{", "background:rgba(255,255,255,.12)!important;", "color:rgba(255,255,255,.82)!important;", "}", "body.sns-chat-page #sns-history-loader-v205.is-loading{", "opacity:.58!important;cursor:default!important;", "}", "@media(max-width:650px){", "body.sns-chat-page #sns-history-loader-v205{", "margin:8px auto 11px!important;", "min-height:23px!important;padding:4px 10px!important;", "font-size:7px!important;", "}", "}" ].join("");
      document.head.appendChild(style);
    })();
    (function() {
      "use strict";
      if (window.__SNS_SEARCH_DOWN_V207__) {
        return;
      }
      window.__SNS_SEARCH_DOWN_V207__ = true;
      function esc(value) {
        return String(value || "").replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;").replace(/'/g, "'");
      }
      function installStyle() {
        if (document.getElementById("sns-search-down-v207-style")) {
          return;
        }
        var style = document.createElement("style");
        style.id = "sns-search-down-v207-style";
        style.textContent = [ "body.sns-chat-page #sns-scroll-bottom-v207{", "position:absolute!important;", "left:50%!important;bottom:142px!important;", "z-index:18000!important;", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:36px!important;height:36px!important;padding:0!important;", "margin-left:-18px!important;", "border:1px solid rgba(255,255,255,.12)!important;border-radius:50%!important;", "background:rgba(255,255,255,.16)!important;color:rgba(255,255,255,.90)!important;", "box-shadow:0 5px 16px rgba(0,0,0,.12)!important;", "backdrop-filter:blur(8px)!important;-webkit-backdrop-filter:blur(8px)!important;", "cursor:pointer!important;", "opacity:0!important;transform:translateY(8px) scale(.92)!important;", "pointer-events:none!important;", "transition:opacity .16s ease,transform .16s ease,background .16s ease,box-shadow .16s ease!important;", "}", "body.sns-chat-page #sns-scroll-bottom-v207.is-visible{", "opacity:1!important;transform:translateY(0) scale(1)!important;", "pointer-events:auto!important;", "}", "body.sns-chat-page #sns-scroll-bottom-v207:hover{background:rgba(255,255,255,.25)!important;box-shadow:0 6px 18px rgba(0,0,0,.15)!important;}", "body.sns-chat-page #sns-scroll-bottom-v207 svg{", "width:18px!important;height:18px!important;", "fill:none!important;stroke:currentColor!important;stroke-width:2.1!important;", "stroke-linecap:round!important;stroke-linejoin:round!important;", "}", "body.sns-chat-page #sns-search-panel-v207{", "position:absolute!important;", "z-index:22000!important;", "top:72px!important;right:17px!important;", "display:none!important;", "box-sizing:border-box!important;", "width:min(365px,calc(100% - 34px))!important;", "max-height:430px!important;", "overflow:hidden!important;", "border:1px solid rgba(70,74,84,.12)!important;border-radius:15px!important;", "background:rgba(247,248,250,.97)!important;", "box-shadow:0 18px 46px rgba(20,22,28,.24)!important;", "backdrop-filter:blur(16px)!important;-webkit-backdrop-filter:blur(16px)!important;", "}", "body.sns-chat-page #sns-search-panel-v207.is-open{display:block!important;}", "body.sns-chat-page .sns-search-v207-head{", "display:grid!important;grid-template-columns:1fr 30px!important;gap:7px!important;", "padding:10px!important;border-bottom:1px solid rgba(70,74,84,.09)!important;", "}", "body.sns-chat-page #sns-search-input-v207{", "box-sizing:border-box!important;width:100%!important;height:34px!important;", "margin:0!important;padding:0 11px!important;", "border:1px solid rgba(70,74,84,.14)!important;border-radius:10px!important;", "background:rgba(255,255,255,.94)!important;color:#222!important;outline:none!important;", "font:12px/34px Arial,sans-serif!important;", "}", "body.sns-chat-page #sns-search-input-v207:focus{", "border-color:rgba(75,82,96,.30)!important;", "box-shadow:0 0 0 3px rgba(75,82,96,.06)!important;", "}", "body.sns-chat-page #sns-search-close-v207{", "display:flex!important;align-items:center!important;justify-content:center!important;", "width:30px!important;height:30px!important;margin:2px 0 0!important;padding:0!important;", "border:0!important;border-radius:50%!important;background:transparent!important;", "color:#666!important;cursor:pointer!important;font:300 22px/1 Arial,sans-serif!important;", "}", "body.sns-chat-page #sns-search-close-v207:hover{background:rgba(0,0,0,.05)!important;}", "body.sns-chat-page #sns-search-status-v207{", "padding:7px 11px 5px!important;color:#8a8a8a!important;", "font:600 8px/1.25 Arial,sans-serif!important;", "letter-spacing:.25px!important;text-transform:uppercase!important;", "}", "body.sns-chat-page #sns-search-results-v207{", "max-height:340px!important;overflow:auto!important;padding:3px 7px 8px!important;", "scrollbar-width:thin!important;", "}", "body.sns-chat-page .sns-search-result-v207{", "display:block!important;width:100%!important;box-sizing:border-box!important;", "margin:0!important;padding:9px 9px!important;text-align:left!important;", "border:0!important;border-radius:10px!important;background:transparent!important;", "color:#222!important;cursor:pointer!important;", "}", "body.sns-chat-page .sns-search-result-v207:hover{background:rgba(0,0,0,.045)!important;}", "body.sns-chat-page .sns-search-result-author-v207{", "display:block!important;margin:0 0 3px!important;color:#555!important;", "font:700 9px/1.2 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-search-result-text-v207{", "display:-webkit-box!important;-webkit-line-clamp:2!important;-webkit-box-orient:vertical!important;", "overflow:hidden!important;color:#333!important;", "font:11px/1.35 Arial,sans-serif!important;", "}", "body.sns-chat-page .sns-search-result-v207 mark{", "padding:0!important;background:rgba(255,213,79,.42)!important;color:inherit!important;", "}", "body.sns-chat-page #pun-viewtopic .sns-message.sns-search-target .post-content{", "animation:snsSearchTargetPulseV207 1.75s ease both!important;", "}", "@keyframes snsSearchTargetPulseV207{", "0%{box-shadow:0 0 0 0 rgba(199,145,66,.00);}", "18%{box-shadow:0 0 0 4px rgba(199,145,66,.28),0 8px 22px rgba(70,55,35,.12);}", "100%{box-shadow:0 0 0 0 rgba(199,145,66,.00);}", "}", "@media(max-width:650px){", "body.sns-chat-page #sns-scroll-bottom-v207{left:50%!important;right:auto!important;bottom:124px!important;width:34px!important;height:34px!important;margin-left:-17px!important;}", "body.sns-chat-page #sns-search-panel-v207{top:66px!important;right:9px!important;width:calc(100% - 18px)!important;max-height:390px!important;}", "body.sns-chat-page #sns-search-results-v207{max-height:300px!important;}", "}" ].join("");
        document.head.appendChild(style);
      }
      function topicNode() {
        return document.querySelector("#sns-chat-shell > .topic");
      }
      function shellNode() {
        return document.getElementById("sns-chat-shell");
      }
      function ensureDownButton() {
        var shell = shellNode();
        if (!shell) {
          return null;
        }
        var button = document.getElementById("sns-scroll-bottom-v207");
        if (!button) {
          button = document.createElement("button");
          button.type = "button";
          button.id = "sns-scroll-bottom-v207";
          button.title = "\u0412\u043d\u0438\u0437";
          button.setAttribute("aria-label", "\u041f\u0435\u0440\u0435\u0439\u0442\u0438 \u043a \u043f\u043e\u0441\u043b\u0435\u0434\u043d\u0438\u043c \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f\u043c");
          button.innerHTML = '<svg viewBox="0 0 24 24" aria-hidden="true">' + '<path d="M6 9l6 6 6-6"></path>' + "</svg>";
          shell.appendChild(button);
        }
        return button;
      }
      function nearBottom(topic) {
        if (!topic) {
          return true;
        }
        return topic.scrollHeight - topic.scrollTop - topic.clientHeight <= 105;
      }
      function updateDownButton() {
        var topic = topicNode();
        var button = ensureDownButton();
        if (!topic || !button) {
          return;
        }
        button.classList.toggle("is-visible", !nearBottom(topic));
      }
      function clearSearchTemp() {
        if (typeof window.__SNS_HISTORY_CLEAR_SEARCH_TEMP__ === "function") {
          window.__SNS_HISTORY_CLEAR_SEARCH_TEMP__();
        }
      }
      function goBottom() {
        var topic = topicNode();
        if (!topic) {
          return;
        }
        clearSearchTemp();
        window.setTimeout(function() {
          topic.scrollTo({
            top: topic.scrollHeight,
            behavior: "smooth"
          });
          window.setTimeout(updateDownButton, 260);
        }, 10);
      }
      function ensureSearchPanel() {
        var shell = shellNode();
        if (!shell) {
          return null;
        }
        var panel = document.getElementById("sns-search-panel-v207");
        if (panel) {
          return panel;
        }
        panel = document.createElement("div");
        panel.id = "sns-search-panel-v207";
        panel.innerHTML = '<div class="sns-search-v207-head">' + '<input id="sns-search-input-v207" type="search" autocomplete="off" placeholder="\u041f\u043e\u0438\u0441\u043a \u043f\u043e \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f\u043c\u2026">' + '<button type="button" id="sns-search-close-v207" title="\u0417\u0430\u043a\u0440\u044b\u0442\u044c" aria-label="\u0417\u0430\u043a\u0440\u044b\u0442\u044c">\xd7</button>' + "</div>" + '<div id="sns-search-status-v207">\u0412\u0432\u0435\u0434\u0438\u0442\u0435 \u0442\u0435\u043a\u0441\u0442 \u0434\u043b\u044f \u043f\u043e\u0438\u0441\u043a\u0430</div>' + '<div id="sns-search-results-v207"></div>';
        shell.appendChild(panel);
        return panel;
      }
      function searchIndex() {
        if (typeof window.__SNS_HISTORY_SEARCH_INDEX__ === "function") {
          try {
            return window.__SNS_HISTORY_SEARCH_INDEX__() || [];
          } catch (error) {
            return [];
          }
        }
        var items = [];
        document.querySelectorAll("#sns-chat-shell>.topic>.post.sns-message").forEach(function(post) {
          var content = post.querySelector(".post-content");
          if (!content) {
            return;
          }
          var value = String(content.textContent || "").replace(/\s+/g, " ").trim();
          if (!value) {
            return;
          }
          var author = post.querySelector(".sns-author-name");
          items.push({
            postId: post.id || "",
            page: Number(window.__SNS_LAZY_HISTORY_MAX_PAGE__ || 1),
            author: author ? String(author.textContent || "").trim() : "",
            text: value
          });
        });
        return items;
      }
      function highlightSnippet(text, query) {
        text = String(text || "");
        query = String(query || "");
        var lower = text.toLocaleLowerCase();
        var qLower = query.toLocaleLowerCase();
        var at = lower.indexOf(qLower);
        if (at < 0) {
          return esc(text.slice(0, 150));
        }
        var start = Math.max(0, at - 48);
        var end = Math.min(text.length, at + query.length + 82);
        var before = text.slice(start, at);
        var hit = text.slice(at, at + query.length);
        var after = text.slice(at + query.length, end);
        return (start > 0 ? "\u2026" : "") + esc(before) + "<mark>" + esc(hit) + "</mark>" + esc(after) + (end < text.length ? "\u2026" : "");
      }
      var searchTimer = null;
      function runSearch() {
        var input = document.getElementById("sns-search-input-v207");
        var status = document.getElementById("sns-search-status-v207");
        var resultsBox = document.getElementById("sns-search-results-v207");
        if (!input || !status || !resultsBox) {
          return;
        }
        var query = String(input.value || "").replace(/\s+/g, " ").trim();
        resultsBox.innerHTML = "";
        if (query.length < 2) {
          status.textContent = query.length ? "\u0412\u0432\u0435\u0434\u0438\u0442\u0435 \u043c\u0438\u043d\u0438\u043c\u0443\u043c 2 \u0441\u0438\u043c\u0432\u043e\u043b\u0430" : "\u0412\u0432\u0435\u0434\u0438\u0442\u0435 \u0442\u0435\u043a\u0441\u0442 \u0434\u043b\u044f \u043f\u043e\u0438\u0441\u043a\u0430";
          return;
        }
        var lowerQuery = query.toLocaleLowerCase();
        var matches = searchIndex().filter(function(item) {
          return (typeof item.lowerText === "string" ? item.lowerText : String(item.text || "").toLocaleLowerCase()).indexOf(lowerQuery) !== -1;
        });
        status.textContent = matches.length ? "\u041d\u0430\u0439\u0434\u0435\u043d\u043e: " + matches.length : "\u0421\u043e\u0432\u043f\u0430\u0434\u0435\u043d\u0438\u0439 \u043d\u0435\u0442";
        matches.slice(0, 80).forEach(function(item) {
          var button = document.createElement("button");
          button.type = "button";
          button.className = "sns-search-result-v207";
          button.setAttribute("data-post-id", String(item.postId || ""));
          button.setAttribute("data-page", String(item.page || 1));
          button.innerHTML = '<span class="sns-search-result-author-v207">' + esc(item.author || "\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435") + "</span>" + '<span class="sns-search-result-text-v207">' + highlightSnippet(item.text, query) + "</span>";
          resultsBox.appendChild(button);
        });
        if (matches.length > 80) {
          var limitNote = document.createElement("div");
          limitNote.style.cssText = "padding:7px 10px 11px;color:#999;font:9px/1.3 Arial,sans-serif;text-align:center;";
          limitNote.textContent = "\u041f\u043e\u043a\u0430\u0437\u0430\u043d\u044b \u043f\u0435\u0440\u0432\u044b\u0435 80 \u0441\u043e\u0432\u043f\u0430\u0434\u0435\u043d\u0438\u0439";
          resultsBox.appendChild(limitNote);
        }
      }
      function scheduleSearchRefresh() {
        var panel = document.getElementById("sns-search-panel-v207");
        if (!panel || !panel.classList.contains("is-open")) return;
        clearTimeout(searchTimer);
        searchTimer = window.setTimeout(runSearch, 120);
      }
      function openSearch() {
        var panel = ensureSearchPanel();
        if (!panel) {
          return;
        }
        panel.classList.add("is-open");
        runSearch();
        var input = document.getElementById("sns-search-input-v207");
        window.setTimeout(function() {
          if (input) {
            input.focus();
            input.select();
          }
        }, 35);
      }
      function closeSearch() {
        var panel = document.getElementById("sns-search-panel-v207");
        if (panel) {
          panel.classList.remove("is-open");
        }
      }
      installStyle();
      $(document).off(".snsSearchDownV207").on("click.snsSearchDownV207", "#sns-scroll-bottom-v207", function(event) {
        event.preventDefault();
        goBottom();
      }).on("click.snsSearchDownV207", ".sns-search-open", function(event) {
        event.preventDefault();
        event.stopPropagation();
        openSearch();
      }).on("click.snsSearchDownV207", "#sns-search-close-v207", function(event) {
        event.preventDefault();
        closeSearch();
      }).on("input.snsSearchDownV207", "#sns-search-input-v207", function() {
        clearTimeout(searchTimer);
        searchTimer = window.setTimeout(runSearch, 120);
      }).on("keydown.snsSearchDownV207", "#sns-search-input-v207", function(event) {
        if (event.key === "Escape") {
          event.preventDefault();
          closeSearch();
        }
      }).on("click.snsSearchDownV207", ".sns-search-result-v207", function(event) {
        event.preventDefault();
        var postId = String(this.getAttribute("data-post-id") || "");
        var page = Number(this.getAttribute("data-page") || 1);
        if (typeof window.__SNS_HISTORY_SHOW_SEARCH_POST__ === "function") {
          window.__SNS_HISTORY_SHOW_SEARCH_POST__(page, postId);
        } else {
          var post = document.getElementById(postId);
          if (post) {
            post.scrollIntoView({
              behavior: "smooth",
              block: "center"
            });
          }
        }
        closeSearch();
        window.setTimeout(updateDownButton, 160);
      }).on("sns_pages_loaded.snsSearchDownV207 sns_initial_history_ready.snsSearchDownV207", function() {
        ensureDownButton();
        updateDownButton();
        scheduleSearchRefresh();
      }).on("pun_post.snsSearchDownV207 pun_edit.snsSearchDownV207 sns_search_index_changed.snsSearchDownV207", scheduleSearchRefresh);
      function bindTopicScroll() {
        var topic = topicNode();
        if (!topic) {
          return;
        }
        if (topic.getAttribute("data-sns-down-bound-v207") === "1") {
          return;
        }
        topic.setAttribute("data-sns-down-bound-v207", "1");
        topic.addEventListener("scroll", updateDownButton, {
          passive: true
        });
      }
      function boot() {
        ensureDownButton();
        ensureSearchPanel();
        bindTopicScroll();
        updateDownButton();
      }
      if (document.readyState === "loading") {
        document.addEventListener("DOMContentLoaded", boot, {
          once: true
        });
      } else {
        boot();
      }
      window.setTimeout(boot, 300);
      window.setTimeout(boot, 1200);
    })();
    (function() {
      "use strict";
      if (document.getElementById("sns-attach-menu-light-v212-style")) {
        return;
      }
      var style = document.createElement("style");
      style.id = "sns-attach-menu-light-v212-style";
      style.textContent = [ "body.sns-chat-page #sns-attach-menu{", "background:rgba(247,248,250,.97)!important;", "color:#2b2d32!important;", "border:1px solid rgba(70,74,84,.12)!important;", "box-shadow:0 12px 32px rgba(20,22,28,.16)!important;", "backdrop-filter:blur(14px)!important;", "-webkit-backdrop-filter:blur(14px)!important;", "}", "body.sns-chat-page #sns-attach-menu button{", "background:transparent!important;", "color:#3a3d44!important;", "}", "body.sns-chat-page #sns-attach-menu button:hover{", "background:rgba(70,74,84,.08)!important;", "color:#202228!important;", "}", "body.sns-chat-page #sns-attach-menu .sns-attach-icon{", "color:#5d626c!important;", "opacity:.9!important;", "}" ].join("");
      document.head.appendChild(style);
    })();
    (function() {
      "use strict";
      window.SNSPhysicalDebug = function() {
        var result = {
          build: "v2.29",
          maxPage: window.__SNS_PHYSICAL_HISTORY_MAX_PAGE__ || window.__SNS_LAZY_HISTORY_MAX_PAGE__ || null,
          oldestLoadedPage: window.__SNS_LAZY_HISTORY_OLDEST_PAGE__ || null,
          lastPhysicalFetch: window.__SNS_LAST_PHYSICAL_FETCH__ || null,
          domPosts: document.querySelectorAll("#sns-chat-shell>.topic>.post").length,
          visibleMessages: document.querySelectorAll("#sns-chat-shell>.topic>.post.sns-message").length
        };
        console.log("SNS PHYSICAL DEBUG", result);
        return result;
      };
    })();
    (function() {
      "use strict";
      if (document.getElementById("sns-v231-compositor-style")) {
        return;
      }
      var style = document.createElement("style");
      style.id = "sns-v231-compositor-style";
      style.textContent = [ "body.sns-chat-page #sns-chat-shell>.sns-dialog-bg-layer{", "backface-visibility:hidden!important;", "-webkit-backface-visibility:hidden!important;", "transform:translateZ(0)!important;", "isolation:isolate!important;", "}", "body.sns-chat-page #sns-chat-shell>.sns-dialog-bg-layer>.sns-dialog-bg-image{", "backface-visibility:hidden!important;", "-webkit-backface-visibility:hidden!important;", "transform:translateZ(0) scale(1.04)!important;", "will-change:transform!important;", "}", "body.sns-chat-page #sns-chat-shell>.sns-dialog-bg-layer:after{", "backface-visibility:hidden!important;", "-webkit-backface-visibility:hidden!important;", "transform:translateZ(0)!important;", "}" ].join("");
      document.head.appendChild(style);
    })();
    (function() {
      "use strict";
      window.SNSLazyState = function() {
        var topic = document.querySelector("#sns-chat-shell>.topic");
        var result = {
          build: "v2.42",
          maxPage: window.__SNS_LAZY_HISTORY_MAX_PAGE__ || null,
          oldestLoadedPage: window.__SNS_LAZY_HISTORY_OLDEST_PAGE__ || null,
          domPosts: topic ? topic.querySelectorAll(":scope > .post").length : 0,
          visibleMessages: topic ? topic.querySelectorAll(":scope > .post.sns-message").length : 0,
          prepending: window.__SNS_LAZY_HISTORY_PREPENDING__ === true
        };
        console.log("SNS LAZY STATE", result);
        return result;
      };
    })();
    var perfStyle = document.createElement("style");
    perfStyle.id = "sns-v241-performance-style";
    perfStyle.textContent = "#sns-smilies-picker .sns-smilie-more{display:block;margin:8px auto;padding:7px 12px;border:0;border-radius:7px;background:rgba(0,0,0,.06);color:inherit;cursor:pointer;font:11px Arial,sans-serif}#sns-smilies-picker .sns-smilie-more[hidden]{display:none!important}";
    document.head.appendChild(perfStyle);
  }
})(window, document);
