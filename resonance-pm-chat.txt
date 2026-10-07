/* RESONANCE PM CHAT v14.5 — RusFF, 2026-10-07
 * Полный файл для замены прежней версии RESONANCE PM CHAT.
 * Подключение: внешний JavaScript либо содержимое между тегами script в HTML-низ.
 * Удалите прежнее подключение; не устанавливайте одновременно CF Messenger.
 * Сохранены оформление, цитирование, медиа, переключение в обычные ЛС и черновики.
 * Штатный редактор и его плагины работают в собственном same-origin iframe.
 * API проверяется по структуре ответа; при неподдерживаемом API используется HTML.
 * Один POST на штатную отправку. Неопределённый ответ не очищает текст и не повторяется.
 * Черновики хранятся по аккаунту, собеседнику и вкладке с TTL 30 дней.
 * Встроены CSS и DOMPurify (лицензия библиотеки сохранена ниже).
 * Локальные тесты используют фикстуры; настройки и плагины реального форума
 * требуют проверки на вашем RusFF. Секреты и внешние зависимости не нужны.
 */
(function () {
  'use strict';

  function pmNoticeText(text) {
    return /(?:\u043d\u043e\u0432[\u043e\u044b][\u0435\u0435\u0445]|\u043d\u0435\u043f\u0440\u043e\u0447\u0438\u0442\u0430\u043d\u043d[\u043e\u044b][\u0435\u0435\u0445])\s+\u043b\u0438\u0447\u043d.{0,12}\u0441\u043e\u043e\u0431\u0449|\u043f\u0440\u0438\u0441\u043b\u0430\u043b.{0,55}\u043b\u0438\u0447\u043d.{0,15}\u0441\u043e\u043e\u0431\u0449|new\s+private\s+message|messag_theme|notification-pm/i.test(String(text || ''));
  }
  function watchPmNotices(win, doc, onNotice) {
    var seen = new WeakMap();
    function shellFor(node) {
      var candidate = null, branch = null;
      for (var n = node, depth = 0; n && n !== doc.body && depth < 12; branch = n, n = n.parentElement, depth++) {
        if (/^(FORM|TEXTAREA|SCRIPT|STYLE)$/.test(n.tagName) || n.matches('.post-content, .rpmc-bubble, .rpmc-native-form')) break;
        var text = String(n.textContent || '').replace(/\s+/g, ' ').trim();
        if (text.length > 1600) break;
        if (!pmNoticeText(text)) continue;
        if (n.querySelector('[data-rpmc-pm-notice="1"]')) return candidate;
        var name = String(n.id || '') + ' ' + String(n.className || '');
        var stack = n.classList.contains('jGrowl') || /(?:^|[\s_-])(?:notifications|toasts|notify-stack|notification-stack|toast-stack)(?:$|[\s_-])/i.test(name);
        // A shared host may also contain upload errors or later receive them.
        // Keep the host visible and choose only the branch with this PM card.
        if (stack) return branch && pmNoticeText(branch.textContent) ? (candidate || branch) : null;
        var position = '';
        if (typeof win.getComputedStyle === 'function') position = win.getComputedStyle(n).position;
        if (position === 'fixed' || position === 'absolute') {
          var separateCard = branch && Array.from(n.children).some(function (child) {
            if (child === branch || /^(SCRIPT|STYLE)$/.test(child.tagName)) return false;
            var value = String(child.textContent || '').replace(/\s+/g, ' ').trim();
            var childName = String(child.id || '') + ' ' + String(child.className || '');
            return pmNoticeText(value) || (value.length > 4 && /(?:error|success|toast|notif|alert)/i.test(childName));
          });
          return separateCard ? (pmNoticeText(branch.textContent) ? (candidate || branch) : null) : n;
        }
        // A class like notification-content is an inner text block, not the
        // card. Continue up to the actual positioned shell, including its
        // avatar, close button, border and shadow.
        if (n.matches('.jGrowl-notification, .toast, .notification, [role="alert"]') ||
            /(?:^|[\s_-])(?:toast-card|notification-card|pm-card)(?:$|[\s_-])/i.test(name)) candidate = n;
      }
      return candidate;
    }
    function inspect(root) {
      if (!root) return;
      if (root.nodeType !== 1) root = root.parentElement;
      if (!root || /^(SCRIPT|STYLE|LINK|META)$/.test(root.tagName)) return;
      var nodes = root.querySelectorAll ? Array.from(root.querySelectorAll(
        'a[href*="messages.php"], .jGrowl-notification, [class*="toast"], [class*="notification"], [class*="notify"], [role="alert"]')).reverse().concat([root]) : [root];
      var handled = new Set();
      nodes.forEach(function (node) {
        var n = shellFor(node);
        if (!n || handled.has(n)) return;
        handled.add(n);
        var link = n.querySelector('a[href*="messages.php"]');
        var signature = n.textContent + '|' + (link && link.getAttribute('href') || '');
        var fresh = seen.get(n) !== signature; seen.set(n, signature);
        if (n.getAttribute('data-rpmc-pm-notice') !== '1') n.setAttribute('data-rpmc-pm-notice', '1');
        onNotice(n, fresh);
      });
    }
    function scan() { if (doc.body) inspect(doc.body); }
    scan();
    doc.addEventListener('DOMContentLoaded', scan, { once: true });
    if (typeof win.MutationObserver !== 'function') return null;
    var observer = new win.MutationObserver(function (records) {
      var roots = new Set();
      records.forEach(function (record) {
        if (record.type === 'characterData') roots.add(record.target.parentElement);
        else if (record.type === 'attributes') roots.add(record.target);
        else Array.from(record.addedNodes).forEach(function (node) { roots.add(node.nodeType === 1 ? node : node.parentElement); });
      });
      roots.forEach(inspect);
    });
    if (doc.documentElement) observer.observe(doc.documentElement, { childList: true, subtree: true, characterData: true,
      attributes: true, attributeFilter: ['class', 'style', 'hidden'] });
    if (win.addEventListener) win.addEventListener('pagehide', function () { observer.disconnect(); }, { once: true });
    return observer;
  }
  function quietEditorFrame(win, doc) {
    if (doc._rpmcQuiet) return; doc._rpmcQuiet = true;
    if (!win || !doc || win === win.top || win.__rpmcQuietEditor) return;
    win.__rpmcQuietEditor = true;
    // Supported settings of Alex_63 notifications. Change only the child
    // window's own object, never preferences/cookies or the top-level module.
    function disableChildNotifications() {
      try {
        var module = win.notifications;
        if (module && typeof module === 'object' && module !== win.top.notifications) {
          module.enabled = false; module.soundEnabled = false; module.blinkInterval = -1;
        }
        var jq = win.jQuery;
        if (jq && typeof jq.jGrowl === 'function' && !jq.jGrowl._rpmcGuard) {
          var original = jq.jGrowl;
          var guarded = function (message, options) {
            var title = String(options && (options.header || options.theme) || '') + ' ' + String(message || '');
            if (pmNoticeText(title)) return jq();
            return original.apply(this, arguments);
          };
          Object.keys(original).forEach(function (key) { guarded[key] = original[key]; });
          guarded._rpmcGuard = true; jq.jGrowl = guarded;
        }
      } catch (_) {}
      if (doc.head && !doc.getElementById('rpmc-quiet-notifications')) {
        var style = doc.createElement('style'); style.id = 'rpmc-quiet-notifications';
        style.textContent = '.rpmc-pm-notification-hidden{display:none!important}';
        doc.head.appendChild(style);
      }
    }
    disableChildNotifications();
    watchPmNotices(win, doc, function (node, fresh) {
      if (!node.classList.contains('rpmc-pm-notification-hidden')) node.classList.add('rpmc-pm-notification-hidden');
      if (node.style.display !== 'none') node.style.setProperty('display', 'none', 'important');
      try { if (fresh && typeof win.parent.ResonancePmIncomingHint === 'function') win.parent.ResonancePmIncomingHint(); } catch (_) {}
    });
    doc.addEventListener('DOMContentLoaded', disableChildNotifications, { once: true });
    if (win.addEventListener) win.addEventListener('load', disableChildNotifications, true);
    if (typeof win.MutationObserver === 'function') {
      var observer = new win.MutationObserver(function (records) {
        if (records.some(function (record) { return record.addedNodes.length; })) disableChildNotifications();
      });
      if (doc.documentElement) observer.observe(doc.documentElement, { childList: true, subtree: true });
      if (win.addEventListener) win.addEventListener('pagehide', function () { observer.disconnect(); }, { once: true });
    }
  }
  var pageUrl = new URL(location.href);
  if (!/\/messages\.php$/i.test(pageUrl.pathname)) return;
  if (window.self !== window.top && (/^resonance-pm-editor-/.test(window.name) ||
      (window.frameElement && window.frameElement.getAttribute('data-rpmc-editor') === '1'))) {
    quietEditorFrame(window, document);
    function editorDomReady() {
      try {
        if (window.frameElement) window.frameElement.dispatchEvent(new window.parent.Event('rpmc-editor-ready'));
      } catch (_) {}
    }
    if (document.readyState === 'loading') document.addEventListener('DOMContentLoaded', editorDomReady, { once: true });
    else editorDomReady();
    return;
  }
  if (window.__resonancePmV13 || window.__resonancePmV14 || document.getElementById('cf-messenger-root')) return;
  var currentMessageId = Number(pageUrl.searchParams.get('id'));
  if (!Number.isSafeInteger(currentMessageId) || currentMessageId <= 0) currentMessageId = 0;
  window.__resonancePmV14 = true;

  var messagePurifier = (function () { var module = { exports: {} }, exports = module.exports;
/*! @license DOMPurify 3.4.16 | (c) Cure53 and other contributors | Released under the Apache license 2.0 and Mozilla Public License 2.0 | github.com/cure53/DOMPurify/blob/3.4.16/LICENSE */
(function(e,t){typeof exports==`object`&&typeof module<`u`?module.exports=t():typeof define==`function`&&define.amd?define([],t):(e=typeof globalThis<`u`?globalThis:e||self,e.DOMPurify=t())})(this,function(){"use strict";function e(e,t){this.v=e,this.k=t}function t(e,t){(t==null||t>e.length)&&(t=e.length);for(var n=0,r=Array(t);n<t;n++)r[n]=e[n];return r}function n(e){if(Array.isArray(e))return e}function r(e,t){var n=e==null?null:typeof Symbol<`u`&&e[Symbol.iterator]||e[`@@iterator`];if(n!=null){var r,i,a,o,s=[],c=!0,l=!1;try{if(a=(n=n.call(e)).next,t===0){if(Object(n)!==n)return;c=!1}else for(;!(c=(r=a.call(n)).done)&&(s.push(r.value),s.length!==t);c=!0);}catch(e){l=!0,i=e}finally{try{if(!c&&n.return!=null&&(o=n.return(),Object(o)!==o))return}finally{if(l)throw i}}return s}}function i(){throw TypeError(`Invalid attempt to destructure non-iterable instance.
In order to be iterable, non-array objects must have a [Symbol.iterator]() method.`)}
/*! regenerator-runtime -- Copyright (c) 2014-present, Facebook, Inc. -- license (MIT): https://github.com/babel/babel/blob/main/packages/babel-helpers/LICENSE */
function a(e,t){return n(e)||r(e,t)||o(e,t)||i()}function o(e,n){if(e){if(typeof e==`string`)return t(e,n);var r={}.toString.call(e).slice(8,-1);return r===`Object`&&e.constructor&&(r=e.constructor.name),r===`Map`||r===`Set`?Array.from(e):r===`Arguments`||/^(?:Ui|I)nt(?:8|16|32)(?:Clamped)?Array$/.test(r)?t(e,n):void 0}}function s(t){var n,r;function i(n,r){try{var o=t[n](r),s=o.value,c=s instanceof e;Promise.resolve(c?s.v:s).then(function(e){if(c){var r=n===`return`&&s.k?n:`next`;if(!s.k||e.done)return i(r,e);e=t[r](e).value}a(!!o.done,e)},function(e){i(`throw`,e)})}catch(e){a(2,e)}}function a(e,t){e===2?n.reject(t):n.resolve({value:t,done:e}),(n=n.next)?i(n.key,n.arg):r=null}this._invoke=function(e,t){return new Promise(function(a,o){var s={key:e,arg:t,resolve:a,reject:o,next:null};r?r=r.next=s:(n=r=s,i(e,t))})},typeof t.return!=`function`&&(this.return=void 0)}s.prototype[typeof Symbol==`function`&&Symbol.asyncIterator||`@@asyncIterator`]=function(){return this},s.prototype.next=function(e){return this._invoke(`next`,e)},s.prototype.throw=function(e){return this._invoke(`throw`,e)},s.prototype.return=function(e){return this._invoke(`return`,e)};let c=Object.entries,l=Object.setPrototypeOf,u=Object.isFrozen,d=Object.getPrototypeOf,f=Object.getOwnPropertyDescriptor,p=Object.freeze,m=Object.seal,h=Object.create,ee=typeof Reflect<`u`&&Reflect,g=ee.apply,te=ee.construct;p||(p=function(e){return e}),m||(m=function(e){return e}),g||(g=function(e,t){var n=[...arguments].slice(2);return e.apply(t,n)}),te||(te=function(e){return new e(...[...arguments].slice(1))});let _=T(Array.prototype.forEach);Array.prototype.indexOf;let ne=T(Array.prototype.lastIndexOf),re=T(Array.prototype.pop),ie=T(Array.prototype.push);Array.prototype.slice;let ae=T(Array.prototype.splice),v=Array.isArray,oe=T(String.prototype.toLowerCase),se=T(String.prototype.toString),ce=T(String.prototype.match),le=T(String.prototype.replace),ue=T(String.prototype.indexOf),de=T(String.prototype.trim),fe=T(Number.prototype.toString),y=T(Boolean.prototype.toString),b=typeof BigInt>`u`?null:T(BigInt.prototype.toString),pe=typeof Symbol>`u`?null:T(Symbol.prototype.toString),x=T(Object.prototype.hasOwnProperty),S=T(Object.prototype.toString),C=T(RegExp.prototype.test),w=E(TypeError);function T(e){return function(t){t instanceof RegExp&&(t.lastIndex=0);var n=[...arguments].slice(1);return g(e,t,n)}}function E(e){return function(){return te(e,[...arguments])}}function D(e,t){let n=arguments.length>2&&arguments[2]!==void 0?arguments[2]:oe;if(l&&l(e,null),!v(t))return e;let r=t.length;for(;r--;){let i=t[r];if(typeof i==`string`){let e=n(i);e!==i&&(u(t)||(t[r]=e),i=e)}e[i]=!0}return e}function me(e){for(let t=0;t<e.length;t++)x(e,t)||(e[t]=null);return e}function O(e){let t=h(null);for(let r of c(e)){var n=a(r,2);let i=n[0],o=n[1];x(e,i)&&(t[i]=v(o)?me(o):o&&typeof o==`object`&&o.constructor===Object?O(o):o)}return t}function he(e){switch(typeof e){case`string`:return e;case`number`:return fe(e);case`boolean`:return y(e);case`bigint`:return b?b(e):`0`;case`symbol`:return pe?pe(e):`Symbol()`;case`undefined`:return S(e);case`function`:case`object`:{if(e===null)return S(e);let t=e,n=k(t,`toString`);if(typeof n==`function`){let e=n(t);return typeof e==`string`?e:S(e)}return S(e)}default:return S(e)}}function k(e,t){for(;e!==null;){let n=f(e,t);if(n){if(n.get)return T(n.get);if(typeof n.value==`function`)return T(n.value)}e=d(e)}function n(){return null}return n}function ge(e){try{return C(e,``),!0}catch(e){return!1}}let _e=p(/* @__PURE__ */ `a.abbr.acronym.address.area.article.aside.audio.b.bdi.bdo.big.blink.blockquote.body.br.button.canvas.caption.center.cite.code.col.colgroup.content.data.datalist.dd.decorator.del.details.dfn.dialog.dir.div.dl.dt.element.em.fieldset.figcaption.figure.font.footer.form.h1.h2.h3.h4.h5.h6.head.header.hgroup.hr.html.i.img.input.ins.kbd.label.legend.li.main.map.mark.marquee.menu.menuitem.meter.nav.nobr.ol.optgroup.option.output.p.picture.pre.progress.q.rp.rt.ruby.s.samp.search.section.select.shadow.slot.small.source.spacer.span.strike.strong.style.sub.summary.sup.table.tbody.td.template.textarea.tfoot.th.thead.time.tr.track.tt.u.ul.var.video.wbr`.split(`.`)),ve=p(/* @__PURE__ */ `svg.a.altglyph.altglyphdef.altglyphitem.animatecolor.animatemotion.animatetransform.circle.clippath.defs.desc.ellipse.enterkeyhint.exportparts.filter.font.g.glyph.glyphref.hkern.image.inputmode.line.lineargradient.marker.mask.metadata.mpath.part.path.pattern.polygon.polyline.radialgradient.rect.stop.style.switch.symbol.text.textpath.title.tref.tspan.view.vkern`.split(`.`)),ye=p([`feBlend`,`feColorMatrix`,`feComponentTransfer`,`feComposite`,`feConvolveMatrix`,`feDiffuseLighting`,`feDisplacementMap`,`feDistantLight`,`feDropShadow`,`feFlood`,`feFuncA`,`feFuncB`,`feFuncG`,`feFuncR`,`feGaussianBlur`,`feImage`,`feMerge`,`feMergeNode`,`feMorphology`,`feOffset`,`fePointLight`,`feSpecularLighting`,`feSpotLight`,`feTile`,`feTurbulence`]),be=p([`animate`,`color-profile`,`cursor`,`discard`,`font-face`,`font-face-format`,`font-face-name`,`font-face-src`,`font-face-uri`,`foreignobject`,`hatch`,`hatchpath`,`mesh`,`meshgradient`,`meshpatch`,`meshrow`,`missing-glyph`,`script`,`set`,`solidcolor`,`unknown`,`use`]),xe=p(/* @__PURE__ */ `math.menclose.merror.mfenced.mfrac.mglyph.mi.mlabeledtr.mmultiscripts.mn.mo.mover.mpadded.mphantom.mroot.mrow.ms.mspace.msqrt.mstyle.msub.msup.msubsup.mtable.mtd.mtext.mtr.munder.munderover.mprescripts`.split(`.`)),Se=p([`maction`,`maligngroup`,`malignmark`,`mlongdiv`,`mscarries`,`mscarry`,`msgroup`,`mstack`,`msline`,`msrow`,`semantics`,`annotation`,`annotation-xml`,`mprescripts`,`none`]),Ce=p([`#text`]),we=p(/* @__PURE__ */ `accept.action.align.alt.autocapitalize.autocomplete.autopictureinpicture.autoplay.background.bgcolor.border.capture.cellpadding.cellspacing.checked.cite.class.clear.color.cols.colspan.command.commandfor.controls.controlslist.coords.crossorigin.datetime.decoding.default.dir.disabled.disablepictureinpicture.disableremoteplayback.download.draggable.enctype.enterkeyhint.exportparts.face.for.headers.height.hidden.high.href.hreflang.id.inert.inputmode.integrity.ismap.kind.label.lang.list.loading.loop.low.max.maxlength.media.method.min.minlength.multiple.muted.name.nonce.noshade.novalidate.nowrap.open.optimum.part.pattern.placeholder.playsinline.popover.popovertarget.popovertargetaction.poster.preload.pubdate.radiogroup.readonly.rel.required.rev.reversed.role.rows.rowspan.spellcheck.scope.selected.shape.size.sizes.slot.span.srclang.start.src.srcset.step.style.summary.tabindex.title.translate.type.usemap.valign.value.width.wrap.xmlns`.split(`.`)),Te=p(/* @__PURE__ */ `accent-height.accumulate.additive.alignment-baseline.amplitude.ascent.attributename.attributetype.azimuth.basefrequency.baseline-shift.begin.bias.by.class.clip.clippathunits.clip-path.clip-rule.color.color-interpolation.color-interpolation-filters.color-profile.color-rendering.cx.cy.d.dx.dy.diffuseconstant.direction.display.divisor.dominant-baseline.dur.edgemode.elevation.end.exponent.fill.fill-opacity.fill-rule.filter.filterunits.flood-color.flood-opacity.font-family.font-size.font-size-adjust.font-stretch.font-style.font-variant.font-weight.fx.fy.g1.g2.glyph-name.glyphref.gradientunits.gradienttransform.height.href.id.image-rendering.in.in2.intercept.k.k1.k2.k3.k4.kerning.keypoints.keysplines.keytimes.lang.lengthadjust.letter-spacing.kernelmatrix.kernelunitlength.lighting-color.local.marker-end.marker-mid.marker-start.markerheight.markerunits.markerwidth.maskcontentunits.maskunits.max.mask.mask-type.media.method.mode.min.name.numoctaves.offset.operator.opacity.order.orient.orientation.origin.overflow.paint-order.path.pathlength.patterncontentunits.patterntransform.patternunits.pointer-events.points.preservealpha.preserveaspectratio.primitiveunits.r.rx.ry.radius.refx.refy.repeatcount.repeatdur.restart.result.rotate.scale.seed.shape-rendering.slope.specularconstant.specularexponent.spreadmethod.startoffset.stddeviation.stitchtiles.stop-color.stop-opacity.stroke-dasharray.stroke-dashoffset.stroke-linecap.stroke-linejoin.stroke-miterlimit.stroke-opacity.stroke.stroke-width.style.surfacescale.systemlanguage.tabindex.tablevalues.targetx.targety.transform.transform-origin.text-anchor.text-decoration.text-orientation.text-rendering.textlength.type.u1.u2.unicode.values.vector-effect.viewbox.visibility.version.vert-adv-y.vert-origin-x.vert-origin-y.width.word-spacing.wrap.writing-mode.xchannelselector.ychannelselector.x.x1.x2.xmlns.y.y1.y2.z.zoomandpan`.split(`.`)),Ee=p(/* @__PURE__ */ `accent.accentunder.align.bevelled.close.columnalign.columnlines.columnspacing.columnspan.denomalign.depth.dir.display.displaystyle.encoding.fence.frame.height.href.id.largeop.length.linethickness.lquote.lspace.mathbackground.mathcolor.mathsize.mathvariant.maxsize.minsize.movablelimits.notation.numalign.open.rowalign.rowlines.rowspacing.rowspan.rspace.rquote.scriptlevel.scriptminsize.scriptsizemultiplier.selection.separator.separators.stretchy.subscriptshift.supscriptshift.symmetric.voffset.width.xmlns`.split(`.`)),De=p([`xlink:href`,`xml:id`,`xlink:title`,`xml:space`,`xmlns:xlink`]),Oe=m(/{{[\w\W]*|^[\w\W]*}}/g),ke=m(/<%[\w\W]*|^[\w\W]*%>/g),Ae=m(/\${[\w\W]*/g),je=m(/^data-[\-\w.\u00B7-\uFFFF]+$/),Me=m(/^aria-[\-\w]+$/),Ne=m(/^(?:(?:(?:f|ht)tps?|mailto|tel|callto|sms|cid|xmpp|matrix):|[^a-z]|[a-z+.\-]+(?:[^a-z+.\-:]|$))/i),Pe=m(/^(?:\w+script|data):/i),Fe=m(/[\u0000-\u0020\u00A0\u1680\u180E\u2000-\u2029\u205F\u3000]/g),Ie=m(/^html$/i),Le=m(/^[a-z][.\w]*(-[.\w]+)+$/i),Re=m(/<[/\w!]/g),ze=m(/<[/\w]/g),Be=m(/<\/no(script|embed|frames)/i),Ve=m(/\/>/i),A={element:1,attribute:2,text:3,cdataSection:4,entityReference:5,entityNode:6,processingInstruction:7,comment:8,document:9,documentType:10,documentFragment:11,notation:12},j=[`style`,`script`,`xmp`,`iframe`,`noembed`,`noframes`,`plaintext`,`noscript`],He=p(D({},j)),Ue=function(){let e={};return _(j,t=>{e[t]=m(RegExp(`</`+t+`(?=[\\t\\n\\f\\r />])`,`i`))}),p(e)}(),We=function(){return typeof window>`u`?null:window},Ge=function(e,t){if(typeof e!=`object`||typeof e.createPolicy!=`function`)return null;let n=null,r=`data-tt-policy-suffix`;t&&t.hasAttribute(r)&&(n=t.getAttribute(r));let i=`dompurify`+(n?`#`+n:``);try{return e.createPolicy(i,{createHTML(e){return e},createScriptURL(e){return e}})}catch(e){return console.warn(`TrustedTypes policy `+i+` could not be created.`),null}},Ke=function(){return{afterSanitizeAttributes:[],afterSanitizeElements:[],afterSanitizeShadowDOM:[],beforeSanitizeAttributes:[],beforeSanitizeElements:[],beforeSanitizeShadowDOM:[],uponSanitizeAttribute:[],uponSanitizeElement:[],uponSanitizeShadowNode:[]}},M=function(e,t,n,r){return x(e,t)&&v(e[t])?D(r.base?O(r.base):{},e[t],r.transform):n},qe=function(e,t,n){let r=x(e,t)?e[t]:void 0;return r&&typeof r==`object`?O(r):n()};function Je(){let e=arguments.length>0&&arguments[0]!==void 0?arguments[0]:We(),t=e=>Je(e);if(t.version=`3.4.16`,t.removed=[],!e||!e.document||e.document.nodeType!==A.document||!e.Element)return t.isSupported=!1,t;let n=e.document,r=n,i=r.currentScript;e.DocumentFragment;let a=e.HTMLTemplateElement,o=e.Node,s=e.Element,l=e.NodeFilter;e.NamedNodeMap===void 0&&(e.NamedNodeMap||e.MozNamedAttrMap),e.HTMLFormElement;let u=e.DOMParser,d=e.trustedTypes,f=s.prototype,ee=k(f,`cloneNode`),g=k(f,`remove`),te=k(f,`removeAttributeNode`),fe=k(f,`nextSibling`),y=k(f,`childNodes`),b=k(f,`parentNode`),pe=k(f,`shadowRoot`),S=k(f,`attributes`),T=o&&o.prototype?k(o.prototype,`nodeType`):null,E=o&&o.prototype?k(o.prototype,`nodeName`):null,me=o&&o.prototype?k(o.prototype,`ownerDocument`):null,j=function(e){return T?T(e):e.nodeType},Ye=function(e){return E?E(e):e.nodeName};if(typeof a==`function`){let e=n.createElement(`template`);e.content&&e.content.ownerDocument&&(n=e.content.ownerDocument)}let N,P=``,Xe,Ze=!1,Qe=0,$e=function(){if(Qe>0)throw w(`A configured TRUSTED_TYPES_POLICY callback (createHTML or createScriptURL) must not call DOMPurify.sanitize, as that causes infinite recursion. Do not pass a policy whose callbacks wrap DOMPurify as TRUSTED_TYPES_POLICY; see the "DOMPurify and Trusted Types" section of the README.`)},F=function(e){$e(),Qe++;try{return N.createHTML(e)}finally{Qe--}},et=function(e){$e(),Qe++;try{return N.createScriptURL(e)}finally{Qe--}},tt=function(){return Ze||(Xe=Ge(d,i),Ze=!0),Xe},nt=n,rt=nt.implementation,it=nt.createNodeIterator,at=nt.createDocumentFragment,ot=nt.getElementsByTagName,st=r.importNode,I=Ke();t.isSupported=typeof c==`function`&&typeof b==`function`&&rt&&rt.createHTMLDocument!==void 0;let ct=Oe,lt=ke,ut=Ae,dt=je,ft=Me,pt=Pe,mt=Fe,ht=Le,gt=Ne,L=null,_t=D({},[..._e,...ve,...ye,...xe,...Ce]),R=null,vt=D({},[...we,...Te,...Ee,...De]),z=Object.seal(h(null,{tagNameCheck:{writable:!0,configurable:!1,enumerable:!0,value:null},attributeNameCheck:{writable:!0,configurable:!1,enumerable:!0,value:null},allowCustomizedBuiltInElements:{writable:!0,configurable:!1,enumerable:!0,value:!1}})),yt=null,bt=null,B=Object.seal(h(null,{tagCheck:{writable:!0,configurable:!1,enumerable:!0,value:null},attributeCheck:{writable:!0,configurable:!1,enumerable:!0,value:null}})),xt=!0,St=!0,Ct=!1,wt=!0,V=!1,H=!0,U=!1,Tt=!1,Et=null,Dt=null,Ot=!1,W=!1,kt=!1,At=!1,jt=!0,Mt=!1,Nt=`user-content-`,Pt=!0,Ft=!1,G={},K=null,It=D({},/* @__PURE__ */ `annotation-xml.audio.colgroup.desc.foreignobject.head.iframe.math.mi.mn.mo.ms.mtext.noembed.noframes.noscript.plaintext.script.selectedcontent.style.svg.template.thead.title.video.xmp`.split(`.`)),Lt=null,Rt=D({},[`audio`,`video`,`img`,`source`,`image`,`track`]),zt=null,Bt=D({},[`alt`,`class`,`for`,`id`,`label`,`name`,`pattern`,`placeholder`,`role`,`summary`,`title`,`value`,`style`,`xmlns`]),Vt=`http://www.w3.org/1998/Math/MathML`,Ht=`http://www.w3.org/2000/svg`,q=`http://www.w3.org/1999/xhtml`,J=q,Ut=!1,Wt=null,Gt=D({},[Vt,Ht,q],se),Kt=p([`mi`,`mo`,`mn`,`ms`,`mtext`]),qt=D({},Kt),Jt=p([`annotation-xml`]),Yt=D({},Jt),Xt=D({},[`title`,`style`,`font`,`a`,`script`]),Zt=null,Qt=[`application/xhtml+xml`,`text/html`],Y=null,$t=null,en=n.createElement(`form`),tn=function(e){return e instanceof RegExp||e instanceof Function},nn=function(){let e=arguments.length>0&&arguments[0]!==void 0?arguments[0]:{};if($t&&$t===e)return;(!e||typeof e!=`object`)&&(e={}),e=O(e),Zt=Qt.indexOf(e.PARSER_MEDIA_TYPE)===-1?`text/html`:e.PARSER_MEDIA_TYPE,Y=Zt===`application/xhtml+xml`?se:oe,L=M(e,`ALLOWED_TAGS`,_t,{transform:Y}),R=M(e,`ALLOWED_ATTR`,vt,{transform:Y}),Wt=M(e,`ALLOWED_NAMESPACES`,Gt,{transform:se}),zt=M(e,`ADD_URI_SAFE_ATTR`,Bt,{transform:Y,base:Bt}),Lt=M(e,`ADD_DATA_URI_TAGS`,Rt,{transform:Y,base:Rt}),K=M(e,`FORBID_CONTENTS`,It,{transform:Y}),yt=M(e,`FORBID_TAGS`,O({}),{transform:Y}),bt=M(e,`FORBID_ATTR`,O({}),{transform:Y}),G=x(e,`USE_PROFILES`)?e.USE_PROFILES&&typeof e.USE_PROFILES==`object`?O(e.USE_PROFILES):e.USE_PROFILES:!1,xt=e.ALLOW_ARIA_ATTR!==!1,St=e.ALLOW_DATA_ATTR!==!1,Ct=e.ALLOW_UNKNOWN_PROTOCOLS||!1,wt=e.ALLOW_SELF_CLOSE_IN_ATTR!==!1,V=e.SAFE_FOR_TEMPLATES||!1,H=e.SAFE_FOR_XML!==!1,U=e.WHOLE_DOCUMENT||!1,W=e.RETURN_DOM||!1,kt=e.RETURN_DOM_FRAGMENT||!1,At=e.RETURN_TRUSTED_TYPE||!1,Ot=e.FORCE_BODY||!1,jt=e.SANITIZE_DOM!==!1,Mt=e.SANITIZE_NAMED_PROPS||!1,Pt=e.KEEP_CONTENT!==!1,Ft=e.IN_PLACE||!1,gt=ge(e.ALLOWED_URI_REGEXP)?e.ALLOWED_URI_REGEXP:Ne,J=typeof e.NAMESPACE==`string`?e.NAMESPACE:q,qt=qe(e,`MATHML_TEXT_INTEGRATION_POINTS`,()=>D({},Kt)),Yt=qe(e,`HTML_INTEGRATION_POINTS`,()=>D({},Jt));let t=qe(e,`CUSTOM_ELEMENT_HANDLING`,()=>h(null));if(z=h(null),x(t,`tagNameCheck`)&&tn(t.tagNameCheck)&&(z.tagNameCheck=t.tagNameCheck),x(t,`attributeNameCheck`)&&tn(t.attributeNameCheck)&&(z.attributeNameCheck=t.attributeNameCheck),x(t,`allowCustomizedBuiltInElements`)&&typeof t.allowCustomizedBuiltInElements==`boolean`&&(z.allowCustomizedBuiltInElements=t.allowCustomizedBuiltInElements),m(z),V&&(St=!1),kt&&(W=!0),G&&(L=D({},Ce),R=h(null),G.html===!0&&(D(L,_e),D(R,we)),G.svg===!0&&(D(L,ve),D(R,Te),D(R,De)),G.svgFilters===!0&&(D(L,ye),D(R,Te),D(R,De)),G.mathMl===!0&&(D(L,xe),D(R,Ee),D(R,De))),B.tagCheck=null,B.attributeCheck=null,x(e,`ADD_TAGS`)&&(typeof e.ADD_TAGS==`function`?B.tagCheck=e.ADD_TAGS:v(e.ADD_TAGS)&&(L===_t&&(L=O(L)),D(L,e.ADD_TAGS,Y))),x(e,`ADD_ATTR`)&&(typeof e.ADD_ATTR==`function`?B.attributeCheck=e.ADD_ATTR:v(e.ADD_ATTR)&&(R===vt&&(R=O(R)),D(R,e.ADD_ATTR,Y))),x(e,`ADD_FORBID_CONTENTS`)&&v(e.ADD_FORBID_CONTENTS)&&(K===It&&(K=O(K)),D(K,e.ADD_FORBID_CONTENTS,Y)),Pt&&(L[`#text`]=!0),U&&D(L,[`html`,`head`,`body`]),L.table&&(D(L,[`tbody`]),delete yt.tbody),e.TRUSTED_TYPES_POLICY){if(typeof e.TRUSTED_TYPES_POLICY.createHTML!=`function`)throw w(`TRUSTED_TYPES_POLICY configuration option must provide a "createHTML" hook.`);if(typeof e.TRUSTED_TYPES_POLICY.createScriptURL!=`function`)throw w(`TRUSTED_TYPES_POLICY configuration option must provide a "createScriptURL" hook.`);let t=N;N=e.TRUSTED_TYPES_POLICY;try{P=F(``)}catch(e){throw N=t,e}}else e.TRUSTED_TYPES_POLICY===null?(N=void 0,P=``):(N===void 0&&(N=tt()),N&&typeof P==`string`&&(P=F(``)));p&&p(e),$t=e},rn=D({},[...ve,...ye,...be]),an=D({},[...xe,...Se]),on=function(e,t,n){return t.namespaceURI===q?e===`svg`:t.namespaceURI===Vt?e===`svg`&&(n===`annotation-xml`||qt[n]):!!rn[e]},sn=function(e,t,n){return t.namespaceURI===q?e===`math`:t.namespaceURI===Ht?e===`math`&&Yt[n]:!!an[e]},cn=function(e,t,n){return t.namespaceURI===Ht&&!Yt[n]||t.namespaceURI===Vt&&!qt[n]?!1:!an[e]&&(Xt[e]||!rn[e])},ln=function(e){let t=b(e);(!t||!t.tagName)&&(t={namespaceURI:J,tagName:`template`});let n=oe(e.tagName),r=oe(t.tagName);return Wt[e.namespaceURI]?e.namespaceURI===Ht?on(n,t,r):e.namespaceURI===Vt?sn(n,t,r):e.namespaceURI===q?cn(n,t,r):!!(Zt===`application/xhtml+xml`&&Wt[e.namespaceURI]):!1},X=function(e){ie(t.removed,{element:e});try{b(e).removeChild(e)}catch(t){if(g(e),!b(e))throw w(`a node selected for removal could not be detached from its tree and cannot be safely returned; refusing to sanitize in place`)}},un=function(e,t,n){try{te(e,t)}catch(t){try{e.removeAttribute(n)}catch(e){}}},dn=function(e){pn(e);let t=y(e);if(t){let e=[];_(t,t=>{ie(e,t)}),_(e,e=>{try{g(e)}catch(e){}})}let n=S(e);if(n)for(let t=n.length-1;t>=0;--t){let r=n[t],i=r&&r.name;typeof i==`string`&&un(e,r,i)}},Z=function(e,n,r){if(!r)try{r=n.getAttributeNode(e)}catch(e){r=null}ie(t.removed,{attribute:r||null,from:n});try{r?te(n,r):n.removeAttribute(e)}catch(t){try{n.removeAttribute(e)}catch(e){}}if(e===`is`){if(W||kt)try{X(n)}catch(e){}else try{n.setAttribute(e,``)}catch(e){}}},fn=function(e){let t=S(e);if(t)for(let n=t.length-1;n>=0;--n){let r=t[n],i=r&&r.name;typeof i!=`string`||R[Y(i)]||un(e,r,i)}},pn=function(e){let t=[e];for(;t.length>0;){let e=t.pop();j(e)===A.element&&fn(e);let n=y(e);if(n)for(let e=n.length-1;e>=0;--e)t.push(n[e])}},mn=function(e,t){return H?e===`patchsrc`||e===`for`&&t!==`label`&&t!==`output`:!1},hn=function(e){if(!H)return;let t=[e];for(;t.length>0;){let e=t.pop(),n=j(e);if(n===A.processingInstruction||n===A.comment&&C(ze,e.data)){try{g(e)}catch(e){}continue}if(n===A.element){let t=e,n=Y(Ye(e));try{t.hasAttribute&&t.hasAttribute(`patchsrc`)&&t.removeAttribute(`patchsrc`),t.hasAttribute&&t.hasAttribute(`for`)&&mn(`for`,n)&&t.removeAttribute(`for`)}catch(e){}}let r=y(e);if(r)for(let e=r.length-1;e>=0;--e)t.push(r[e])}},gn=function(e){let t=null,r=null;if(Ot)e=`<remove></remove>`+e;else{let t=ce(e,/^[\r\n\t ]+/);r=t&&t[0]}Zt===`application/xhtml+xml`&&J===q&&(e=`<html xmlns="http://www.w3.org/1999/xhtml"><head></head><body>`+e+`</body></html>`);let i=N?F(e):e;if(J===q)try{t=new u().parseFromString(i,Zt)}catch(e){}if(!t||!t.documentElement){t=rt.createDocument(J,`template`,null);try{t.documentElement.innerHTML=Ut?P:i}catch(e){}}let a=t.body||t.documentElement;return e&&r&&a.insertBefore(n.createTextNode(r),a.childNodes[0]||null),J===q?ot.call(t,U?`html`:`body`)[0]:U?t.documentElement:a},_n=function(e){let t=me?me(e):e.ownerDocument;return it.call(t||e,e,l.SHOW_ELEMENT|l.SHOW_COMMENT|l.SHOW_TEXT|l.SHOW_PROCESSING_INSTRUCTION|l.SHOW_CDATA_SECTION,null)},vn=function(e){return e=le(e,ct,` `),e=le(e,lt,` `),e=le(e,ut,` `),e},yn=function(e){var t;bn(e);let n=(t=e.querySelectorAll)==null?void 0:t.call(e,`template`);n&&_(n,e=>{Sn(e.content)&&yn(e.content)})},bn=function(e){e.normalize();let t=me?me(e):e.ownerDocument,n=it.call(t||e,e,l.SHOW_TEXT|l.SHOW_COMMENT|l.SHOW_CDATA_SECTION|l.SHOW_PROCESSING_INSTRUCTION,null),r=n.nextNode();for(;r;)r.data=vn(r.data),r=n.nextNode()},xn=function(e){let t=E?E(e):null;return typeof t!=`string`||Y(t)!==`form`?!1:typeof e.nodeName!=`string`||typeof e.textContent!=`string`||typeof e.removeChild!=`function`||e.attributes!==S(e)||typeof e.removeAttribute!=`function`||typeof e.removeAttributeNode!=`function`||typeof e.getAttributeNode!=`function`||typeof e.setAttribute!=`function`||typeof e.namespaceURI!=`string`||typeof e.insertBefore!=`function`||typeof e.hasChildNodes!=`function`||e.nodeType!==T(e)||e.childNodes!==y(e)},Sn=function(e){if(!T||typeof e!=`object`||!e)return!1;try{return T(e)===A.documentFragment}catch(e){return!1}},Cn=function(e){if(!T||typeof e!=`object`||!e)return!1;try{return typeof T(e)==`number`}catch(e){return!1}};function Q(e,n,r){e.length!==0&&_(e,e=>{e.call(t,n,r,$t)})}let wn=function(e,t){return!!(H&&e.hasChildNodes()&&!Cn(e.firstElementChild)&&C(Re,e.textContent)&&C(Re,e.innerHTML)||H&&e.namespaceURI===q&&He[t]&&(Cn(e.firstElementChild)||typeof e.textContent==`string`&&C(Ue[t],e.textContent))||e.nodeType===A.processingInstruction||H&&e.nodeType===A.comment&&C(ze,e.data))},Tn=function(e,t){return e instanceof RegExp?C(e,t):e instanceof Function&&!!e(t,...[...arguments].slice(2))},En=function(e,t,n){if(!yt[t]&&jn(t)&&Tn(z.tagNameCheck,t))return!1;if(Pt&&!K[t]){let t=b(e),r=y(e);if(r&&t){let i=r.length;for(let a=i-1;a>=0;--a){let i=e===n?ee(r[a],!0):r[a];t.insertBefore(i,fe(e))}}}return X(e),!0},Dn=function(e,t,n,r){return e.length===0?t:t===n||t===r?O(t):t},$=function(e,t){return e===t||b(e)!==null?!1:(Ft&&pn(e),!0)},On=function(e,n){if(Q(I.beforeSanitizeElements,e,null),$(e,n))return!0;if(xn(e))return X(e),!0;let r=Y(Ye(e));if(L=Dn(I.uponSanitizeElement,L,_t,Et),Q(I.uponSanitizeElement,e,{tagName:r,allowedTags:L}),$(e,n))return!0;if(wn(e,r))return X(e),!0;if(yt[r]||!(B.tagCheck instanceof Function&&B.tagCheck(r))&&!L[r]){let t=En(e,r,n);return t===!1&&(Q(I.afterSanitizeElements,e,null),$(e,n))?!0:t}if(j(e)===A.element&&!ln(e)||(r===`noscript`||r===`noembed`||r===`noframes`)&&C(Be,e.innerHTML))return X(e),!0;if(V&&e.nodeType===A.text){let n=vn(e.textContent);e.textContent!==n&&(ie(t.removed,{element:e.cloneNode()}),e.textContent=n)}return Q(I.afterSanitizeElements,e,null),$(e,n)},kn=function(e,t,r){if(bt[t]||mn(t,e)||jt&&(t===`id`||t===`name`)&&(r in n||r in en))return!1;let i=R[t]||B.attributeCheck instanceof Function&&B.attributeCheck(t,e);return St&&C(dt,t)||xt&&C(ft,t)?!0:i?zt[t]||C(gt,le(r,mt,``))||(t===`src`||t===`xlink:href`||t===`href`)&&e!==`script`&&ue(r,`data:`)===0&&Lt[e]||Ct&&!C(pt,le(r,mt,``))?!0:!r:jn(e)&&Tn(z.tagNameCheck,e)&&Tn(z.attributeNameCheck,t,e)||t===`is`&&z.allowCustomizedBuiltInElements&&Tn(z.tagNameCheck,r)},An=D({},[`annotation-xml`,`color-profile`,`font-face`,`font-face-format`,`font-face-name`,`font-face-src`,`font-face-uri`,`missing-glyph`]),jn=function(e){return!An[oe(e)]&&C(ht,e)},Mn=function(e,t,n,r){if(N&&typeof d==`object`&&typeof d.getAttributeType==`function`&&!n)switch(d.getAttributeType(e,t)){case`TrustedHTML`:return F(r);case`TrustedScriptURL`:return et(r)}return r},Nn=function(e,t,n,r){try{return n?e.setAttributeNS(n,t,r):e.setAttribute(t,r),!xn(e)||(X(e),!1)}catch(n){return Z(t,e),!1}},Pn=function(e,n){if(Q(I.beforeSanitizeAttributes,e,null),$(e,n))return;let r=e.attributes;if(!r||xn(e))return;R=Dn(I.uponSanitizeAttribute,R,vt,Dt);let i={attrName:``,attrValue:``,keepAttr:!0,allowedAttributes:R,forceKeepAttr:void 0},a=r.length,o=Y(e.nodeName);for(;a--;){let n=r[a],s=n.name,c=n.namespaceURI,l=n.value,u=Y(s),d=l,f=s===`value`?d:de(d),p=!1;if(i.attrName=u,i.attrValue=f,i.keepAttr=!0,i.forceKeepAttr=void 0,Q(I.uponSanitizeAttribute,e,i),f=i.attrValue,Mt&&(u===`id`||u===`name`)&&ue(f,Nt)!==0&&(Z(s,e,n),f=Nt+f,p=!0),H&&C(/((--!?|])>)|<\/(style|script|title|xmp|textarea|noscript|iframe|noembed|noframes)/i,f)){Z(s,e,n);continue}if(u===`attributename`&&ce(f,`href`)){Z(s,e,n);continue}if(!i.forceKeepAttr){if(!i.keepAttr){Z(s,e,n);continue}if(!wt&&C(Ve,f)){Z(s,e,n);continue}if(V&&(f=vn(f)),!kn(o,u,f)){Z(s,e,n);continue}f=Mn(o,u,c,f),f!==d&&Nn(e,s,c,f)&&p&&re(t.removed)}}Q(I.afterSanitizeAttributes,e,null),$(e,n)},Fn=function(e){let t=null,n=_n(e);for(Q(I.beforeSanitizeShadowDOM,e,null);t=n.nextNode();)if(Q(I.uponSanitizeShadowNode,t,null),On(t,e),Pn(t,e),Sn(t.content)&&Fn(t.content),j(t)===A.element){let e=pe(t);Sn(e)&&(In(e),Fn(e))}V&&bn(e),Q(I.afterSanitizeShadowDOM,e,null)},In=function(e){let t=[{node:e,shadow:null}];for(;t.length>0;){let e=t.pop();if(e.shadow){Fn(e.shadow);continue}let n=e.node,r=j(n)===A.element,i=y(n);if(i)for(let e=i.length-1;e>=0;--e)t.push({node:i[e],shadow:null});if(r){let e=E?E(n):null;if(typeof e==`string`&&Y(e)===`template`){let e=n.content;Sn(e)&&t.push({node:e,shadow:null})}}if(r){let e=pe(n);Sn(e)&&t.push({node:null,shadow:e},{node:e,shadow:null})}}};return t.sanitize=function(e){let n=arguments.length>1&&arguments[1]!==void 0?arguments[1]:{},i=null,a=null,o=null,s=null;if(Ut=!e,Ut&&(e=`<!-->`),typeof e!=`string`&&!Cn(e)&&(e=he(e),typeof e!=`string`))throw w(`dirty is not a string, aborting`);if(!t.isSupported)return e;Tt?(L=Et,R=Dt):nn(n),(I.uponSanitizeElement.length>0||I.uponSanitizeAttribute.length>0)&&(L=O(L)),I.uponSanitizeAttribute.length>0&&(R=O(R)),t.removed=[];let c=Ft&&typeof e!=`string`&&Cn(e);if(c){hn(e);let t=Ye(e);if(typeof t==`string`){let n=Y(t);if(!L[n]||yt[n])throw dn(e),w(`root node is forbidden and cannot be sanitized in-place`)}if(xn(e))throw dn(e),w(`root node is clobbered and cannot be sanitized in-place`);try{In(e)}catch(t){throw dn(e),t}}else if(Cn(e))i=gn(`<!---->`),a=i.ownerDocument.importNode(e,!0),a.nodeType===A.element&&a.nodeName===`BODY`||a.nodeName===`HTML`?i=a:i.appendChild(a),In(i);else{if(!W&&!V&&!U&&e.indexOf(`<`)===-1)return N&&At?F(e):e;if(i=gn(e),!i)return W?null:At?P:``}i&&Ot&&X(i.firstChild);let l=c?e:i;try{let e=_n(l);for(;o=e.nextNode();)On(o,l),Pn(o,l),Sn(o.content)&&Fn(o.content)}catch(n){throw c&&(dn(e),_(t.removed,e=>{e.element&&pn(e.element)})),n}if(c){let n=!1;if(_(t.removed,t=>{t.element&&(t.element===e&&(n=!0),pn(t.element))}),n)throw w(`a node selected for removal could not be safely returned; refusing to sanitize in place`);return V&&yn(e),e}if(W){if(V&&yn(i),kt)for(s=at.call(i.ownerDocument);i.firstChild;)s.appendChild(i.firstChild);else s=i;return(R.shadowroot||R.shadowrootmode)&&(s=st.call(r,s,!0)),s}let u=U?i.outerHTML:i.innerHTML;return U&&L[`!doctype`]&&i.ownerDocument&&i.ownerDocument.doctype&&i.ownerDocument.doctype.name&&C(Ie,i.ownerDocument.doctype.name)&&(u=`<!DOCTYPE `+i.ownerDocument.doctype.name+`>
`+u),V&&(u=vn(u)),N&&At?F(u):u},t.setConfig=function(){let e=arguments.length>0&&arguments[0]!==void 0?arguments[0]:{};nn(e),Tt=!0,Et=L,Dt=R},t.clearConfig=function(){$t=null,Tt=!1,Et=null,Dt=null,N=Xe,P=``},t.isValidAttribute=function(e,t,n){$t||nn({});let r=Y(e),i=Y(t);return kn(r,i,n)},t.addHook=function(e,t){typeof t==`function`&&x(I,e)&&ie(I[e],t)},t.removeHook=function(e,t){if(x(I,e)){if(t!==void 0){let n=ne(I[e],t);return n===-1?void 0:ae(I[e],n,1)[0]}return re(I[e])}},t.removeHooks=function(e){x(I,e)&&(I[e]=[])},t.removeAllHooks=function(){I=Ke()},t}return Je()});

 return module.exports; })();

  var currentBox = pageUrl.searchParams.get('box') || '0';
  var currentPage = Math.max(1, Number(pageUrl.searchParams.get('p')) || 1);
  var memory = new Map();
  var state = { rows: [], warnings: [], partner: null, chat: null, hidden: [],
    refresh: null, draft: '', pending: null, frame: null, editorReady: false,
    subject: '', frameGeneration: 0, submitting: false, notice: '', selectedQuote: null, queuedQuotes: [],
    mailboxRows: [], mailboxReady: false, mailboxJob: null, list: null, mode: 'chat',
    readIds: new Set(), pollTimer: null, pollBusy: false, pollErrors: 0, stopped: false,
    sendFrame: null, sendSequence: 0, unreadOnly: false, listFilter: '', hadPartialHistory: false,
    outgoing: [], outgoingSequence: 0, confirmedOutgoingIds: new Set(),
    avatarQueue: [], avatarWorkers: 0, avatarObserver: null,
    mailboxPaging: {}, pollRound: 0, historySync: null, autoHistory: false,
    initializing: false, current: null, historyError: '',
    lastPoll: 0, cacheTimer: null, pollStarted: false, incomingHint: false, lastHint: 0,
    updatesChannel: null, noticeObserver: null };
  var avatars = new Map(), avatarJobs = new Map();
  var POLL_INTERVAL = 15000;
  var NOTICE_POLL_GAP = 5000;
  var REQUEST_GAP = 500;
  var net = { queue: [], jobs: new Map(), cache: new Map(), running: false,
    lastStart: 0, blockedUntil: 0, failures: 0 };
  var editorNavigation = 0;
  var EDITOR_POPUPS = '#smilies-area, #font-area, #size-area, #color-area, #addition-area, #image-area, #video-area, #table-area';
  var $ = function (s, r) { return (r || document).querySelector(s); };
  var $$ = function (s, r) { return Array.from((r || document).querySelectorAll(s)); };
  var normalizeText = function (v) { return String(v || '').replace(/\s+/g, ' ').trim(); };
  function esc(v) { return String(v == null ? '' : v).replace(/[&<>"']/g, function (c) {
    return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c];
  }); }

  var draftTimer = null, draftDirty = false, draftSaved = false, draftRevision = 0;
  var draftTab = Math.random().toString(36).slice(2) + Date.now().toString(36);
  try { draftTab = sessionStorage.getItem('rpmc-draft-tab') || draftTab; sessionStorage.setItem('rpmc-draft-tab', draftTab); } catch (_) {}
  var DRAFT_TTL = 30 * 86400000, DISPLAY_LIMIT = 500;
  window.addEventListener('pagehide', saveDraftNow);
  window.addEventListener('beforeunload', saveDraftNow);
  state.displayCount = 30; state.displayBefore = null; state.folderSnapshots = {};
  state.tombstones = new Set(); state.historyBusy = false; state.olderRound = 0;
  function draftPrefix() {
    var account = Number(window.UserID);
    return Number.isSafeInteger(account) && account > 0 && state.partner && (state.partner.id || state.partner.name)
      ? 'rpmc-v14-draft:' + account + ':' + (state.partner.id || 'name-' + encodeURIComponent(state.partner.name)) + ':' : '';
  }
  function editorAdapter(frame, textarea) {
    function editorWindow() {
      if (frame && frame.contentWindow) return frame.contentWindow;
      if (frame && frame.Event) return frame;
      return window;
    }
    function makeEvent(type) {
      var win = editorWindow();
      try { return new win.Event(type, { bubbles: true }); }
      catch (_) { return new Event(type, { bubbles: true }); }
    }
    function wysi() {
      var win = editorWindow();
      var api = win && win.WYSI;
      try { return api && typeof api.isActive === 'function' && api.isActive() ? api : null; }
      catch (_) { return null; }
    }
    return {
      getDraft: function () {
        var api = wysi();
        if (api) {
          if (typeof api.getBBCode !== 'function') throw new Error('Визуальный редактор не позволяет прочитать BBCode. Переключись в текстовый режим.');
          return String(api.getBBCode() || '');
        }
        return String(textarea.value || '');
      },
      setDraft: function (text) {
        var api = wysi();
        if (api) {
          if (typeof api.setBBCode !== 'function') throw new Error('Визуальный редактор не позволяет восстановить текст. Переключись в текстовый режим.');
          api.setBBCode(text);
        }
        textarea.value = text;
        textarea.dispatchEvent(makeEvent('input'));
      },
      flushToTextarea: function () { textarea.value = this.getDraft(); return textarea.value; },
      insertQuote: function (addition) {
        var value = this.getDraft();
        if (wysi()) this.setDraft(value + (value ? '\n\n' : '') + addition);
        else {
          var start = textarea.selectionStart == null ? value.length : textarea.selectionStart;
          var end = textarea.selectionEnd == null ? start : textarea.selectionEnd;
          var insert = (start && value[start - 1] !== '\n' ? '\n\n' : '') + addition;
          textarea.setRangeText(insert, start, end, 'end');
          textarea.dispatchEvent(makeEvent('input'));
        }
      }
    };
  }
  function simpleEditorAdapter(textarea) {
    return {
      getDraft: function () { return String(textarea && textarea.value || ''); },
      setDraft: function (text) {
        if (!textarea) return;
        textarea.value = String(text || '');
        textarea.dispatchEvent(new Event('input', { bubbles: true }));
      },
      flushToTextarea: function () { return String(textarea && textarea.value || ''); },
      insertQuote: function (addition) {
        if (!textarea) return;
        var value = String(textarea.value || '');
        var start = textarea.selectionStart == null ? value.length : textarea.selectionStart;
        var end = textarea.selectionEnd == null ? start : textarea.selectionEnd;
        var insert = (start && value[start - 1] !== '\n' ? '\n\n' : '') + addition;
        textarea.setRangeText(insert, start, end, 'end');
        textarea.dispatchEvent(new Event('input', { bubbles: true }));
      }
    };
  }
  function captureDraft() {
    try {
      var simple = state.chat && $('.rpmc-simple-textarea', state.chat);
      if (simple) {
        var simpleText = String(simple.value || '');
        if (state.draft !== simpleText) { state.draft = simpleText; draftRevision++; draftDirty = true; }
        return;
      }
      var inlineFound = state.editorReady && state.chat && postForm(state.chat);
      if (inlineFound && inlineFound.form.classList.contains('rpmc-native-inline')) {
        var inlineText = editorAdapter(window, inlineFound.textarea).getDraft();
        if (state.draft !== inlineText) { state.draft = inlineText; draftRevision++; draftDirty = true; }
        return;
      }
      var found = state.editorReady && state.frame && postForm(state.frame.contentDocument);
      if (found) {
        var text = editorAdapter(state.frame, found.textarea).getDraft();
        if (state.draft !== text) { state.draft = text; draftRevision++; draftDirty = true; }
      }
    } catch (_) { /* Keep the last readable snapshot; submission reports adapter errors. */ }
  }
  function saveDraftNow() {
    clearTimeout(draftTimer); draftTimer = null;
    captureDraft();
    var prefix = draftPrefix(); if (!prefix) return false;
    var operations = state.outgoing.map(function (item) {
      return { id: item.id, text: item.text, status: item.status === 'sending' ? 'unknown' : item.status,
        maxId: item.maxId, before: Array.from(item.before || []), started: item.started, serverId: item.serverId || 0 };
    });
    try {
      localStorage.setItem(prefix + draftTab, JSON.stringify({ schema: 14, updated: Date.now(),
        expires: Date.now() + DRAFT_TTL, revision: draftRevision, text: state.draft, operations: operations }));
      draftSaved = true; draftDirty = false;
    } catch (_) { draftSaved = false; }
    var node = state.chat && $('.rpmc-draft-state', state.chat);
    if (node) node.textContent = draftSaved ? '' : '\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u0441\u043e\u0445\u0440\u0430\u043d\u0438\u0442\u044c \u0447\u0435\u0440\u043d\u043e\u0432\u0438\u043a \u0432 \u0431\u0440\u0430\u0443\u0437\u0435\u0440\u0435. \u0421\u043a\u043e\u043f\u0438\u0440\u0443\u0439 \u0442\u0435\u043a\u0441\u0442 \u043f\u0435\u0440\u0435\u0434 \u0437\u0430\u043a\u0440\u044b\u0442\u0438\u0435\u043c \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u044b.';
    return draftSaved;
  }
  function scheduleDraftSave() {
    captureDraft(); draftDirty = true; clearTimeout(draftTimer);
    draftTimer = setTimeout(saveDraftNow, 400);
  }
  function restoreDraft() {
    var prefix = draftPrefix(); if (!prefix) return;
    try {
      var best = null, own = null;
      for (var i = localStorage.length - 1; i >= 0; i--) {
        var key = localStorage.key(i);
        if (!key || key.indexOf('rpmc-v14-draft:') !== 0) continue;
        var value;
        try { value = JSON.parse(localStorage.getItem(key)); } catch (_) { continue; }
        if (!value || value.schema !== 14 || !Number.isFinite(value.expires) || value.expires <= Date.now()) { localStorage.removeItem(key); continue; }
        if (key.indexOf(prefix) !== 0 || typeof value.text !== 'string') continue;
        if (key === prefix + draftTab) own = value;
        if (!best || value.updated > best.updated) best = value;
      }
      var record = own || best; if (!record) return;
      state.draft = record.text; draftRevision = Number(record.revision) || 0;
      state.outgoing = (Array.isArray(record.operations) ? record.operations : []).filter(function (item) {
        return item && typeof item.text === 'string' && Array.isArray(item.before) && /^(unknown|sent|failed)$/.test(item.status);
      }).map(function (item) { return Object.assign({}, item, { before: new Set(item.before), node: null }); });
      state.notice = state.outgoing.some(function (item) { return item.status === 'unknown'; })
        ? '\u0412\u043e\u0441\u0441\u0442\u0430\u043d\u043e\u0432\u043b\u0435\u043d \u0442\u0435\u043a\u0441\u0442 \u0441 \u043d\u0435\u043f\u043e\u0434\u0442\u0432\u0435\u0440\u0436\u0434\u0435\u043d\u043d\u043e\u0439 \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u043e\u0439. \u041f\u0440\u043e\u0432\u0435\u0440\u044c \u043f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0443 \u043f\u0435\u0440\u0435\u0434 \u043f\u043e\u0432\u0442\u043e\u0440\u043e\u043c.' : '';
    } catch (_) { draftSaved = false; }
  }
  function messageTimestamp(text) {
    text = normalizeText(text).replace(/\s*#\d+\s*$/, '');
    if (/^\d{4}-\d{2}-\d{2}/.test(text)) { var iso = Date.parse(text); if (Number.isFinite(iso)) return iso; }
    var clock = text.match(/(\d{1,2}):(\d{2})(?::(\d{2}))?/); if (!clock) return 0;
    var now = new Date(), date = text.match(/(\d{1,2})[.\/](\d{1,2})[.\/](\d{4})/);
    if (date) now = new Date(+date[3], +date[2] - 1, +date[1]);
    else if (/\u0432\u0447\u0435\u0440\u0430|yesterday/i.test(text)) now.setDate(now.getDate() - 1);
    else if (!/\u0441\u0435\u0433\u043e\u0434\u043d\u044f|today/i.test(text)) {
      var parsed = Date.parse(text); return Number.isFinite(parsed) ? parsed : 0;
    }
    now.setHours(+clock[1], +clock[2], +(clock[3] || 0), 0); return now.getTime();
  }
  function compareMessages(a, b) { return (Number(a.timestamp) || 0) - (Number(b.timestamp) || 0) || a.id - b.id; }
  function validEntry(e) {
    return e && Number.isSafeInteger(e.id) && e.id > 0 && /^(in|out)$/.test(e.direction) && localUrl(e.href);
  }
  function sanitizeHtml(html) {
    var doc = new DOMParser().parseFromString('<body>' + String(html || '') + '</body>', 'text/html');
    return safeMessageHtml(doc.body);
  }

  function accountCacheKey() {
    var id = Number(window.UserID);
    return Number.isSafeInteger(id) && id > 0 ? 'rpmc-v14-5-cache:' + id : '';
  }
  function restoreCache() {
    var key = accountCacheKey(); if (!key) return;
    try {
      var value = JSON.parse(window.sessionStorage.getItem(key) || 'null');
      if (!value || value.schema !== 16 || value.policy !== 3 || !Number.isFinite(value.expires) || value.expires <= Date.now()) return;
      if (Array.isArray(value.rows)) state.mailboxRows = value.rows.filter(validEntry).slice(0, 3000);
      if (value.paging && typeof value.paging === 'object') state.mailboxPaging = value.historyComplete === false ? {} : value.paging;
      (value.messages || []).slice(-1500).forEach(function (item) {
        if (Array.isArray(item) && typeof item[0] === 'string' && item[1] && typeof item[1].html === 'string') memory.set(item[0], Object.assign({}, item[1], { html: sanitizeHtml(item[1].html) }));
      });
      state.mailboxReady = !!state.mailboxRows.length;
    } catch (_) {}
  }
  function saveCacheNow() {
    clearTimeout(state.cacheTimer); state.cacheTimer = null;
    if (!accountCacheKey()) return;
  try {
    // Serialize once, with a bounded text budget. Repeatedly serializing the
    // whole cache while dropping one item at a time used to block the UI.
    var messages = [], budget = 1800000;
    Array.from(memory.entries()).slice(-1500).reverse().some(function (item) {
      var size = JSON.stringify(item).length;
      if (size > budget) return false;
      messages.push(item); budget -= size; return budget <= 0;
    });
    messages.reverse();
    var value = { schema: 16, policy: 3, historyComplete: state.mailboxRows.length <= 3000, expires: Date.now() + 21600000,
      rows: state.mailboxRows.slice().sort(function (a, b) { return b.id - a.id; }).slice(0, 3000),
      paging: state.mailboxPaging, messages: messages };
    var json = JSON.stringify(value);
    window.sessionStorage.setItem(accountCacheKey(), json);
  } catch (_) {}
  }
  function scheduleCacheSave() {
    if (state.cacheTimer || !accountCacheKey()) return;
    state.cacheTimer = setTimeout(saveCacheNow, 2000);
  }
  function urlOf(href, base) {
    try { return new URL(href, base || location.href); } catch (_) { return null; }
  }
  function localUrl(href, base) {
    var u = urlOf(href, base);
    return u && u.origin === location.origin ? u : null;
  }
  function linkId(a) {
    var u = urlOf(a && a.getAttribute('href'));
    return u ? Number(u.searchParams.get('id')) || 0 : 0;
  }
  function mailboxUrl(box, p) {
    return '/messages.php?box=' + encodeURIComponent(box) + '&p=' + (p || 1);
  }
  function nativeUrl(href) {
    var u = localUrl(href); if (!u) return '#';
    u.searchParams.delete('rpmc_mode');
    u.searchParams.set('rpmc_native', '1'); return u.href;
  }
  function chatUrl(href) {
    var u = localUrl(href); if (!u) return '#';
    u.searchParams.delete('rpmc_native'); u.searchParams.set('rpmc_mode', 'chat'); return u.href;
  }
  function modeKey() {
    return 'resonance-pm-mode:' + String(window.UserID || 'default');
  }
  function preferredMode() {
    try { return window.localStorage.getItem(modeKey()) === 'native' ? 'native' : 'chat'; }
    catch (_) { return 'chat'; }
  }
  function setMode(mode) {
    try { window.localStorage.setItem(modeKey(), mode); } catch (_) {}
  }
  function resolveMode() {
    var chosen = pageUrl.searchParams.get('rpmc_mode');
    if (chosen === 'chat' || chosen === 'native') { setMode(chosen); return chosen; }
    return pageUrl.searchParams.get('rpmc_native') === '1' ? 'native' : preferredMode();
  }
  function modeLink(href, mode) {
    var u = localUrl(href); if (!u) return '#';
    u.searchParams.delete('rpmc_native'); u.searchParams.set('rpmc_mode', mode); return u.href;
  }
  function direction(box) { return box === '1' || box === '2' ? 'out' : 'in'; }
  function ownProfileLinks(root) {
    return $$('a[href*="profile.php"]', root).filter(function (a) {
      var u = localUrl(a.getAttribute('href'));
      return u && /\/profile\.php$/i.test(u.pathname) && linkId(a) > 0 && normalizeText(a.textContent);
    });
  }
  function decodeHtml(buffer, contentType, fallback) {
    var bytes = new Uint8Array(buffer);
    if (bytes[0] === 0xff && bytes[1] === 0xfe) return new TextDecoder('utf-16le').decode(bytes);
    if (bytes[0] === 0xfe && bytes[1] === 0xff) return new TextDecoder('utf-16be').decode(bytes);
    // Valid UTF-8 wins over a stale server header; ASCII is identical in cp1251.
    try { return new TextDecoder('utf-8', { fatal: true }).decode(bytes); } catch (_) {}
    var head = new TextDecoder('windows-1252').decode(bytes.subarray(0, 8192));
    var header = String(contentType || '').match(/charset\s*=\s*["']?([^;\s"']+)/i);
    var meta = head.match(/<meta\b[^>]*charset\s*=\s*["']?\s*([^\s"'/>;]+)/i);
    var encoding = (header && header[1]) || (meta && meta[1]) || fallback || 'windows-1251';
    if (/^utf-?8$/i.test(encoding)) encoding = 'windows-1251';
    try { return new TextDecoder(encoding).decode(bytes); }
    catch (_) { return new TextDecoder('windows-1251').decode(bytes); }
  }
  function isLoginResponse(doc, responseUrl) {
    // A maintenance splash can keep a hidden login form in EVERY response,
    // including an administrator's mailbox and a successful send receipt.
    // Only a real login route or the native main login form means login is needed.
    if (responseUrl && /\/login\.php$/i.test(responseUrl.pathname)) return true;
    if (!doc) return false;
    return $$('form', doc).some(function (form) {
      var action = localUrl(form.getAttribute('action') || '', responseUrl && responseUrl.href);
      if (!action || !/\/login\.php$/i.test(action.pathname) || ! $('input[type="password"]', form)) return false;
      if (form.closest('#resplash, #maintenance-screen, #pircs2, #LogIn_Window')) return false;
      return !!form.closest('#pun-login, #pun-main');
    });
  }
  function readLocalState(key) {
    try { return JSON.parse(window.localStorage.getItem(key) || 'null'); } catch (_) { return null; }
  }
  function writeLocalState(key, value) {
    try { window.localStorage.setItem(key, JSON.stringify(value)); return true; } catch (_) { return false; }
  }
  function pauseRequests(ms) {
    net.blockedUntil = Math.max(net.blockedUntil, Number(readLocalState('rpmc-network-pause-v142')) || 0, Date.now() + ms);
    writeLocalState('rpmc-network-pause-v142', net.blockedUntil);
  }
  function networkPause() {
    return Math.max(net.blockedUntil, Number(readLocalState('rpmc-network-pause-v142')) || 0);
  }
  function waitDelay(ms) { return new Promise(function (resolve) { setTimeout(resolve, Math.max(0, ms)); }); }
  async function networkGate(run) {
    async function execute() {
      if (state.stopped) throw new Error('\u0421\u0442\u0440\u0430\u043d\u0438\u0446\u0430 \u0437\u0430\u043a\u0440\u044b\u0442\u0430');
      if (networkPause() > Date.now()) throw new Error('\u0417\u0430\u043f\u0440\u043e\u0441\u044b \u0432\u0440\u0435\u043c\u0435\u043d\u043d\u043e \u043f\u0440\u0438\u043e\u0441\u0442\u0430\u043d\u043e\u0432\u043b\u0435\u043d\u044b \u043f\u043e\u0441\u043b\u0435 \u043e\u0448\u0438\u0431\u043a\u0438 \u0444\u043e\u0440\u0443\u043c\u0430');
      var last = Math.max(net.lastStart, Number(readLocalState('rpmc-network-start-v142')) || 0);
      var wait = last + REQUEST_GAP - Date.now();
      if (wait > 0) await waitDelay(wait);
      if (state.stopped) throw new Error('\u0421\u0442\u0440\u0430\u043d\u0438\u0446\u0430 \u0437\u0430\u043a\u0440\u044b\u0442\u0430');
      if (networkPause() > Date.now()) throw new Error('\u0417\u0430\u043f\u0440\u043e\u0441\u044b \u0432\u0440\u0435\u043c\u0435\u043d\u043d\u043e \u043f\u0440\u0438\u043e\u0441\u0442\u0430\u043d\u043e\u0432\u043b\u0435\u043d\u044b \u043f\u043e\u0441\u043b\u0435 \u043e\u0448\u0438\u0431\u043a\u0438 \u0444\u043e\u0440\u0443\u043c\u0430');
      net.lastStart = Date.now(); writeLocalState('rpmc-network-start-v142', net.lastStart);
      return run();
    }
    // The same lock serializes THIS chat's reads across tabs on the same forum.
    if (typeof navigator !== 'undefined' && navigator.locks && navigator.locks.request) {
      return navigator.locks.request('resonance-pm-network-v9', execute);
    }
    // Lease fallback for browsers without Web Locks. Local storage is browser
    // storage; these checks never contact the forum API.
    var token = String(Date.now()) + ':' + Math.random().toString(36).slice(2);
    for (var attempt = 0; attempt < 120; attempt++) {
      var lease = readLocalState('rpmc-network-lease-v142');
      if (!lease || !lease.owner || lease.expires <= Date.now()) {
        if (!writeLocalState('rpmc-network-lease-v142', { owner: token, expires: Date.now() + 60000 })) return execute();
        await waitDelay(60);
        lease = readLocalState('rpmc-network-lease-v142');
        if (lease && lease.owner === token) {
          try { return await execute(); }
          finally {
            lease = readLocalState('rpmc-network-lease-v142');
            if (lease && lease.owner === token) writeLocalState('rpmc-network-lease-v142', null);
          }
        }
      }
      await waitDelay(500);
    }
    throw new Error('\u0417\u0430\u043f\u0440\u043e\u0441\u044b \u044d\u0442\u043e\u0439 \u0432\u043a\u043b\u0430\u0434\u043a\u0438 \u043e\u0436\u0438\u0434\u0430\u044e\u0442 \u0437\u0430\u0432\u0435\u0440\u0448\u0435\u043d\u0438\u044f \u0434\u0440\u0443\u0433\u043e\u0439 \u0432\u043a\u043b\u0430\u0434\u043a\u0438');
  }
  function queueNetwork(key, run, priority) {
    if (net.jobs.has(key)) return net.jobs.get(key);
    var resolveJob, rejectJob;
    var promise = new Promise(function (resolve, reject) { resolveJob = resolve; rejectJob = reject; });
    net.jobs.set(key, promise);
    net.queue.push({ key: key, run: run, priority: priority || 0, resolve: resolveJob, reject: rejectJob });
    drainNetwork();
    return promise;
  }
  async function drainNetwork() {
    if (net.running) return;
    net.running = true;
    try {
      while (net.queue.length) {
        net.queue.sort(function (a, b) { return b.priority - a.priority; });
        var job = net.queue.shift();
        try {
          if (state.stopped) throw new Error('\u0421\u0442\u0440\u0430\u043d\u0438\u0446\u0430 \u0437\u0430\u043a\u0440\u044b\u0442\u0430');
          job.resolve(await networkGate(job.run));
        } catch (error) { job.reject(error); }
        finally { net.jobs.delete(job.key); }
      }
    } finally { net.running = false; }
  }
  async function readDocFromServer(u) {
    var isProfile = /\/profile\.php$/i.test(u.pathname);
    var controller = new AbortController();
    var timer = setTimeout(function () { controller.abort(); }, 18000);
    try {
      var requestUrl = new URL(u.href);
      // Read-only fragments do not need the forum header/footer. The real editor
      // still navigates normally and retains all its native scripts and plugins.
      if (/\/(messages|profile)\.php$/i.test(requestUrl.pathname)) requestUrl.searchParams.set('nohead', '1');
      var response = await fetch(requestUrl.href, { credentials: 'same-origin', cache: 'no-store', signal: controller.signal });
      if (!response.ok) {
        var failure = new Error('HTTP ' + response.status);
        if (!isProfile && (response.status === 429 || response.status === 503)) {
          var retry = response.headers.get('retry-after') || '';
          var seconds = /^\d+$/.test(retry) ? Number(retry) * 1000 : Date.parse(retry) - Date.now();
          pauseRequests(Math.max(120000, Number.isFinite(seconds) ? seconds : 0));
        }
        throw failure;
      }
      var responseUrl = localUrl(response.url || u.href);
      if (!responseUrl || /\/login\.php$/i.test(responseUrl.pathname)) throw new Error('\u041d\u0443\u0436\u043d\u043e \u0432\u043e\u0439\u0442\u0438 \u043d\u0430 \u0444\u043e\u0440\u0443\u043c');
      var html = decodeHtml(await response.arrayBuffer(), response.headers.get('content-type'), document.characterSet);
      var doc = new DOMParser().parseFromString(html, 'text/html');
      if (isLoginResponse(doc, responseUrl)) throw new Error('\u041d\u0443\u0436\u043d\u043e \u0432\u043e\u0439\u0442\u0438 \u043d\u0430 \u0444\u043e\u0440\u0443\u043c');
      net.failures = 0; return doc;
    } catch (error) {
      if (isProfile) throw error;
      net.failures++;
      if (networkPause() <= Date.now()) pauseRequests(Math.min(120000, 5000 * Math.pow(3, Math.min(net.failures - 1, 3))));
      throw error;
    } finally { clearTimeout(timer); }
  }
  function fetchDoc(href, options) {
    options = options || {};
    var u = localUrl(href); if (!u) return Promise.reject(new Error('\u0421\u0441\u044b\u043b\u043a\u0430 \u0432\u0435\u0434\u0451\u0442 \u0437\u0430 \u043f\u0440\u0435\u0434\u0435\u043b\u044b \u0444\u043e\u0440\u0443\u043c\u0430'));
    u.hash = ''; u.searchParams.delete('rpmc_mode'); u.searchParams.delete('rpmc_native');
    var key = u.href, cached = net.cache.get(key);
    if (!options.fresh && cached && cached.expires > Date.now()) return Promise.resolve(cached.doc);
    var isProfile = /\/profile\.php$/i.test(u.pathname);
    return queueNetwork(key, async function () {
      var doc = await readDocFromServer(u);
      var ttl = isProfile || u.searchParams.has('id') ? 600000 : 10000;
      net.cache.set(key, { doc: doc, expires: Date.now() + ttl });
      while (net.cache.size > 100) net.cache.delete(net.cache.keys().next().value);
      return doc;
    }, options.priority == null ? (isProfile ? -30 : 10) : options.priority);
  }

  var apiIdentityDisabledUntil = 0;
  async function forumApiGet(method, params, verb) {
    verb = verb || 'GET';
    var query = new URLSearchParams(); query.set('method', method);
    Object.keys(params || {}).forEach(function (key) {
      var value = params[key]; if (value !== undefined && value !== null && value !== '') query.set(key, String(value));
    });
    var isUsers = method === 'users.get';
    return queueNetwork('api:' + verb + ':' + query.toString(), async function () {
      var controller = new AbortController(), timer = setTimeout(function () { controller.abort(); }, 15000);
      try {
        var options = { method: verb, credentials: 'same-origin', cache: 'no-store',
          headers: { 'X-Requested-With': 'XMLHttpRequest' }, signal: controller.signal };
        var href = '/api.php';
        if (verb === 'GET') href += '?' + query.toString();
        else { options.headers['Content-Type'] = 'application/x-www-form-urlencoded; charset=UTF-8'; options.body = query.toString(); }
        var response = await fetch(href, options);
        if (response.status === 429) {
          var retry = response.headers.get('retry-after') || '';
          var ms = /^\d+$/.test(retry) ? Number(retry) * 1000 : Date.parse(retry) - Date.now();
          pauseRequests(Math.max(15000, Number.isFinite(ms) ? ms : 0));
        }
        var text = decodeHtml(await response.arrayBuffer(), response.headers.get('content-type'), document.characterSet), json;
        try { json = JSON.parse(text); } catch (_) { throw new Error('API ' + method + ': ответ не является JSON (HTTP ' + response.status + ').'); }
        if (!response.ok || !json || typeof json !== 'object' || json.error || !Object.prototype.hasOwnProperty.call(json, 'response')) {
          throw new Error('API ' + method + ': запрос отклонён (HTTP ' + response.status + ').');
        }
        return json.response;
      } finally { clearTimeout(timer); }
    }, isUsers ? -20 : 15);
  }

  async function apiLookupMessage(id) {
    if (!Number.isSafeInteger(Number(id)) || Number(id) <= 0 || Date.now() < apiIdentityDisabledUntil) return null;
    try {
      var rows = await forumApiGet('message.get', { id: Number(id), sort_dir: 'desc', limit: 1,
        fields: 'id,subject,message,user_id,username,status,showed,received,posted,datetime' });
      if (!Array.isArray(rows)) return null;
      return rows.find(function (row) { return Number(row.id) === Number(id); }) || null;
    } catch (_) {
      apiIdentityDisabledUntil = Date.now() + 60000;
      return null;
    }
  }

  var API_FIELDS = 'id,subject,message,user_id,username,status,showed,received,posted,datetime';
  var apiState = { history: false, recent: false, historySkip: 0, recentSkip: 0, historyDone: false, recentDone: false,
    historyIds: new Set(), recentIds: new Set(), dialogRows: [], users: new Map(), disabledUntil: 0, readBusy: false, readDisabledUntil: 0 };
  function bindEditorKeys(form, textarea) {
    textarea.addEventListener('keydown', function (event) {
      if ((event.ctrlKey || event.metaKey) && event.key === 'Enter') {
        event.preventDefault(); if (state.submitting) return;
        var button = $$('input[type="submit"], button[type="submit"], button:not([type])', form).find(function (item) { return !isPreview(item); });
        if (button) button.click();
      }
    });
  }
  function bindDirectSubmission(form, adapter, textarea, frame) {
    // Never redefine form.submit: named controls and RusFF plugins can make it
    // nonconfigurable. formdata also fires for HTMLFormElement.submit().
    form.addEventListener('formdata', function (event) {
      var pending = state.pending;
      if (pending && pending.form === form) {
        pending.nativeIssued = true;
        try {
          var actual = adapter.flushToTextarea(); event.formData.set(textarea.name || 'req_message', actual);
          if (state.draft !== actual) { state.draft = actual; draftRevision++; }
          pending.draft = actual; pending.revision = draftRevision;
          if (pending.optimistic) { pending.optimistic.text = actual; var bubble = pending.optimistic.node && $('.rpmc-pending-text', pending.optimistic.node); if (bubble) bubble.textContent = actual; }
          if (pending.button && pending.button.name && !event.formData.has(pending.button.name)) event.formData.set(pending.button.name, pending.button.value);
          saveDraftNow();
        } catch (error) { setStatus(error.message, true); }
        return;
      }
      var text;
      try { text = adapter.flushToTextarea(); } catch (error) { setStatus(error.message, true); return; }
      event.formData.set(textarea.name || 'req_message', text);
      var send = $$('input[type="submit"], button[type="submit"], button:not([type])', form).find(function (button) { return !isPreview(button); });
      if (send && send.name && !event.formData.has(send.name)) event.formData.set(send.name, send.value);
      if (state.draft !== text) { state.draft = text; draftRevision++; }
      pending = { id: draftTab + ':' + Date.now() + ':' + (++state.outgoingSequence), started: Date.now(), stage: 'submitting', kind: 'send',
        before: new Set(state.rows.filter(function (entry) { return entry.direction === 'out'; }).flatMap(sourceIds)),
        maxId: Math.max(0, ...state.rows.flatMap(sourceIds)), draft: text, revision: draftRevision,
        adapter: adapter, form: form, textarea: textarea, editor: frame, nativeIssued: true,
        controls: $$('input, button, textarea, select', form).map(function (node) { return { node: node, disabled: !!node.disabled }; }) };
      var receiver = ensureSendFrame(); receiver._rpmcRequest = pending; pending.receiver = receiver;
      state.pending = pending; form._rpmcLastPending = pending; state.submitting = true;
      startOutgoing(pending); sendFeedback(pending, true); watchSendResponse(pending); form.setAttribute('aria-busy', 'true');
      pending.timer = setTimeout(function () { completeSend(pending, false, 'Ответ сервера не получен. Проверь отправку перед повтором.', true); }, 35000);
      setStatus('Отправляю…'); saveDraftNow();
    });
  }
  function normalizedApiRows(rows, partner) {
    if (!Array.isArray(rows)) throw new Error('API вернул неподдерживаемый список сообщений.');
    var used = new Set();
    return rows.map(function (row) {
      var id = Number(row && row.id), userId = Number(row && row.user_id), status = Number(row && row.status);
      // CF's contract distinguishes sent=1 and incoming=0. Unknown statuses do
      // not silently become incoming; use the native HTML adapter instead.
      if (!row || !Number.isSafeInteger(id) || id <= 0 || used.has(id) ||
          !Number.isSafeInteger(userId) || userId < 0 || row.status == null || ![0, 1].includes(status) ||
          typeof row.message !== 'string') throw new Error('Неподдерживаемые поля API сообщения.');
      if (partner && userId > 0 && userId !== partner.id) throw new Error('API вернул сообщение другого собеседника.');
      used.add(id);
      var person = apiState.users.get(userId), partnerId = partner ? partner.id : userId;
      var name = normalizeText(person && person.username || row.username || partner && partner.name);
      var outgoing = status === 1, stamp = Number(row.posted);
      var body = document.createElement('div'); body.innerHTML = row.message;
      var entry = { id: id, box: outgoing ? '1' : '0', direction: outgoing ? 'out' : 'in',
        href: location.origin + '/messages.php?box=' + (outgoing ? '1' : '0') + '&id=' + id,
        subject: normalizeText(row.subject), partnerId: partnerId, partnerName: name || (partnerId ? 'Собеседник #' + partnerId : 'Собеседник'),
        partnerHref: partnerId ? '/profile.php?id=' + partnerId : '', avatar: person && person.avatar || '',
        dateText: String(row.datetime || ''), date: String(row.datetime || ''),
        timestamp: Number.isFinite(stamp) && stamp > 0 ? stamp * 1000 : messageTimestamp(row.datetime),
        unread: !outgoing && Number(row.showed) !== 1, html: safeMessageHtml(body), api: true,
        preview: normalizeText(body.textContent).slice(0, 180) };
      if (row.num_unshowed != null) entry.unreadCount = Math.max(0, Number(row.num_unshowed) || 0);
      return entry;
    });
  }
  async function fetchApiUsers(ids) {
    ids = Array.from(new Set(ids.filter(function (id) { return id > 0 && !apiState.users.has(id); }))).slice(0, 30);
    if (!ids.length) return;
    try {
      var result = await forumApiGet('users.get', { user_id: ids.join(','), limit: ids.length,
        fields: 'user_id,username,avatar,is_online,last_visit_datetime' });
      if (result && Array.isArray(result.users)) result.users.forEach(function (user) {
        var id = Number(user && user.user_id); if (Number.isSafeInteger(id) && id > 0) apiState.users.set(id, user);
      });
    } catch (_) { /* Optional profile data must not block the mailbox. */ }
  }
  async function apiRecent(more) {
    if (Date.now() < apiState.disabledUntil || (more && apiState.recentDone)) return false;
    var skip = more ? apiState.recentSkip : 0;
    try {
      var raw = await forumApiGet('message.getRecent', { sort_dir: 'desc', skip: skip, limit: 30, fields: API_FIELDS + ',num_unshowed' });
      var entries = normalizedApiRows(raw);
      // A strict schema probe succeeded; enrich names independently.
      await fetchApiUsers(entries.map(function (entry) { return entry.partnerId; }));
      entries = normalizedApiRows(raw);
      if (!more) { apiState.recentIds = new Set(); if (entries.length < 30) apiState.dialogRows = []; }
      var dialogues = new Map(apiState.dialogRows.map(function (entry) { return [entry.partnerId || 'message:' + entry.id, entry]; }));
      entries.forEach(function (entry) { dialogues.set(entry.partnerId || 'message:' + entry.id, entry); });
      apiState.dialogRows = Array.from(dialogues.values());
      entries.forEach(function (entry) { apiState.recentIds.add(entry.id); });
      if (more && entries.length && entries.every(function (entry) { return state.mailboxRows.some(function (old) { return old.api && old.id === entry.id; }); })) {
        throw new Error('API повторил страницу списка диалогов.');
      }
      if (!more && entries.length < 30) state.mailboxRows = entries;
      else {
        var map = new Map(state.mailboxRows.map(function (entry) { return [entry.direction + ':' + entry.id, entry]; }));
        entries.forEach(function (entry) { map.set(entry.direction + ':' + entry.id, entry); });
        state.mailboxRows = Array.from(map.values());
      }
      apiState.recent = true; apiState.recentSkip = skip + entries.length; apiState.recentDone = entries.length < 30;
      if (apiState.recentDone) apiState.dialogRows = apiState.dialogRows.filter(function (entry) { return apiState.recentIds.has(entry.id); });
      state.mailboxReady = true; state.warnings = []; scheduleCacheSave(); return true;
    } catch (error) {
      if (apiState.recent) throw error;
      apiState.disabledUntil = Date.now() + 60000; return false;
    }
  }
  async function apiHistory(more, scrollToEnd) {
    if (!state.partner || !state.partner.id || Date.now() < apiState.disabledUntil || (more && apiState.historyDone)) return false;
    var skip = more ? apiState.historySkip : 0;
    try {
      var raw = await forumApiGet('message.get', { user_id: state.partner.id, sort_dir: 'desc', skip: skip, limit: 30, fields: API_FIELDS });
      var entries = normalizedApiRows(raw, state.partner);
      if (!more) apiState.historyIds = new Set();
      var previous = apiState.historyIds.size;
      entries.forEach(function (entry) { apiState.historyIds.add(entry.id); });
      if (more && entries.length && previous === apiState.historyIds.size) throw new Error('API повторил страницу истории.');
      var ids = new Set(entries.map(function (entry) { return entry.id; }));
      // Reconcile only the authoritative returned range, never erase older
      // messages just because pagination did not include them.
      var boundary = entries.length ? entries[entries.length - 1] : null;
      if (!more) state.rows = state.rows.filter(function (entry) {
        return ids.has(entry.id) || (entries.length === 30 && boundary && compareMessages(entry, boundary) < 0);
      });
      if (entries.length < 30) state.rows = state.rows.filter(function (entry) { return apiState.historyIds.has(entry.id); });
      entries.forEach(function (entry) { if (entry.unread) state.readIds.delete(entry.id); memory.set(entry.direction + ':' + entry.id, { html: entry.html, date: entry.date, timestamp: entry.timestamp }); });
      var map = new Map(state.mailboxRows.map(function (entry) { return [entry.direction + ':' + entry.id, entry]; }));
      entries.forEach(function (entry) { map.set(entry.direction + ':' + entry.id, entry); });
      state.mailboxRows = Array.from(map.values());
      if (entries.length < 30) state.mailboxRows = state.mailboxRows.filter(function (entry) { return !matchesPartner(entry, state.partner) || apiState.historyIds.has(entry.id); });
      apiState.history = true; apiState.historySkip = skip + entries.length; apiState.historyDone = entries.length < 30;
      state.warnings = []; state.historyError = ''; mergeVisible(entries, scrollToEnd); scheduleCacheSave(); return true;
    } catch (error) {
      if (apiState.history) throw error;
      apiState.disabledUntil = Date.now() + 60000; return false;
    }
  }
  async function markVisibleApiRead() {
    if (!apiState.history || apiState.readBusy || Date.now() < apiState.readDisabledUntil || document.hidden || !state.chat) return;
    var area = $('.rpmc-messages', state.chat), bounds = area.getBoundingClientRect();
    var ids = state.rows.filter(function (entry) {
      if (entry.direction !== 'in' || !entry.unread || state.readIds.has(entry.id)) return false;
      var node = $('[data-message-key="in:' + entry.id + '"]', area);
      if (!node) return false;
      var rect = node.getBoundingClientRect(); return rect.bottom > Math.max(0, bounds.top) && rect.top < Math.min(window.innerHeight, bounds.bottom);
    }).map(function (entry) { return entry.id; }).slice(0, 30);
    if (!ids.length) return;
    apiState.readBusy = true;
    try {
      await forumApiGet('message.markAsRead', { id: ids.join(',') }, 'POST');
      ids.forEach(function (id) { state.readIds.add(id); });
      state.mailboxRows.forEach(function (entry) { if (ids.includes(entry.id)) entry.unread = false; });
      scheduleCacheSave();
    } catch (_) { apiState.readDisabledUntil = Date.now() + 60000; }
    finally { apiState.readBusy = false; }
  }
  async function resolveCompanion(current, known) {
    var entry = Object.assign({}, current, known || {}), me = Number(window.UserID);
    var selfAuthor = entry.direction === 'out' && entry.partnerId === me;
    if (entry.partnerId && entry.partnerName && !selfAuthor) return entry;
    // API gives the counterpart for incoming AND outgoing PMs, as in CF.
    var api = await apiLookupMessage(entry.id), id = api && Number(api.user_id);
    if (Number.isSafeInteger(id) && id > 0 && (!selfAuthor || id !== me)) {
      entry.partnerId = id; entry.partnerName = normalizeText(api.username) || (known && known.partnerName) || '';
      entry.partnerHref = '/profile.php?id=' + id;
      await fetchApiUsers([id]);
      var card = apiState.users.get(id); if (card && card.username) entry.partnerName = normalizeText(card.username);
    }
    if ((!entry.partnerId || selfAuthor && entry.partnerId === me) && entry.direction === 'out') {
      var reply = localUrl(replyHref(document));
      var recipient = reply && Number(reply.searchParams.get('uid') || reply.searchParams.get('user_id'));
      if (recipient > 0 && recipient !== me) { entry.partnerId = recipient; entry.partnerHref = '/profile.php?id=' + recipient; entry.partnerName = ''; }
      else {
        var page = await fetchDoc(mailboxUrl(currentBox, currentPage));
        var rows = parseMailboxRows(page, currentBox);
        var row = rows.find(function (item) { return item.id === entry.id; });
        if (row) entry = Object.assign(entry, row);
      }
    }
    if (entry.partnerId && (!entry.partnerName || /^(?:удал[её]нный пользователь|deleted user|собеседник(?: #\d+)?)$/i.test(entry.partnerName))) {
      await fetchApiUsers([entry.partnerId]);
      var user = apiState.users.get(entry.partnerId);
      if (user && user.username) entry.partnerName = normalizeText(user.username);
      else {
        try {
          var profile = await fetchDoc(entry.partnerHref || '/profile.php?id=' + entry.partnerId, { priority: 20 });
          var name = $('.pa-author a, #profile-left strong, #profile-name, .username, #pun-profile h1', profile);
          if (name) entry.partnerName = normalizeText(name.textContent).replace(/^(?:профиль|profile)\s*[:—-]\s*/i, '');
        } catch (_) {}
      }
      if (!entry.partnerName || /удал[её]нный пользователь|deleted user/i.test(entry.partnerName)) entry.partnerName = 'Собеседник #' + entry.partnerId;
    }
    if (entry.direction === 'out' && entry.partnerId === me && !known && (!api || Number(api.user_id) !== me)) {
      throw new Error('Получатель исходящего письма не установлен. Открой диалог через список ЛС.');
    }
    if (!entry.partnerId && !entry.partnerName) throw new Error('Не удалось определить собеседника этого письма');
    return entry;
  }
  function addFallbackToolbar(form, textarea) {
    var toolbar = document.createElement('div'); toolbar.className = 'rpmc-fallback-toolbar'; toolbar.setAttribute('role', 'toolbar'); toolbar.setAttribute('aria-label', 'BBCode');
    [['B','b'],['I','i'],['U','u'],['Цитата','quote'],['Код','code'],['Спойлер','spoiler'],['Ссылка','url'],['Картинка','img']].forEach(function (item) {
      var button = document.createElement('button'); button.type = 'button'; button.textContent = item[0];
      button.addEventListener('click', function () {
        if (state.submitting) return;
        var start = textarea.selectionStart, end = textarea.selectionEnd, selection = textarea.value.slice(start, end);
        var open = '[' + item[1] + ']', close = '[/' + item[1] + ']';
        textarea.setRangeText(open + selection + close, start, end, 'select');
        textarea.focus(); textarea.setSelectionRange(start + open.length, start + open.length + selection.length);
        textarea.dispatchEvent(new Event('input', { bubbles: true }));
      }); toolbar.appendChild(button);
    });
    textarea.before(toolbar);
  }

  function navigateEditor(frame, href) {
    clearTimeout(frame._rpmcLoadTimer); frame._rpmcInitialUrl = href;
    frame._rpmcLoadTimer = setTimeout(function () {
      if (frame === state.frame && !state.editorReady) editorFallback(frame, href, '\u0420\u0435\u0434\u0430\u043a\u0442\u043e\u0440 \u043d\u0435 \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u043b\u0441\u044f \u0437\u0430 35 \u0441\u0435\u043a\u0443\u043d\u0434. \u041c\u043e\u0436\u043d\u043e \u043f\u043e\u0432\u0442\u043e\u0440\u0438\u0442\u044c \u0437\u0430\u0433\u0440\u0443\u0437\u043a\u0443.');
    }, 35000);
    // The visible native editor is critical UI. Do not put its GET behind the
    // mailbox/profile backoff queue: a failed history/avatar request must not
    // leave the composer hidden. This mirrors the working CF Messenger approach.
    var u = localUrl(href);
    if (!u) {
      editorFallback(frame, href, '\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043e\u0442\u043a\u0440\u044b\u0442\u044c \u0440\u0435\u0434\u0430\u043a\u0442\u043e\u0440.');
      return Promise.resolve();
    }
    u.searchParams.delete('rpmc_mode');
    u.searchParams.delete('rpmc_native');
    try {
      if (frame === state.frame && frame.isConnected) frame.src = u.href;
      return Promise.resolve();
    } catch (error) {
      editorFallback(frame, href, error.message || '\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043e\u0442\u043a\u0440\u044b\u0442\u044c \u0440\u0435\u0434\u0430\u043a\u0442\u043e\u0440.');
      return Promise.resolve();
    }
  }

  async function mapLimit(items, limit, fn) {
    var cursor = 0, out = new Array(items.length);
    async function worker() { while (cursor < items.length) {
      var i = cursor++; out[i] = await fn(items[i], i);
    } }
    await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
    return out;
  }

  function parseMailboxRows(doc, box) {
    var result = [], seen = new Set();
    var scope = $('#pun-messages', doc) || $('#pun-main', doc) || doc.body;
    if (!scope) throw new Error('\u041d\u0435 \u0440\u0430\u0441\u043f\u043e\u0437\u043d\u0430\u043d\u0430 \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u0430 \u043f\u0430\u043f\u043a\u0438 \u041b\u0421');

    // RusFF themes differ a lot. Do not require one specific table/class: use
    // message links as the contract and then climb to the nearest row/card.
    var messageLinks = $$('a[href*="messages.php"]', scope).filter(function (a) {
      var u = localUrl(a.getAttribute('href'));
      if (!u || !/\/messages\.php$/i.test(u.pathname) || u.searchParams.get('action')) return false;
      var id = Number(u.searchParams.get('id'));
      return Number.isSafeInteger(id) && id > 0;
    });

    messageLinks.forEach(function (subjectLink) {
      var u = localUrl(subjectLink.getAttribute('href'));
      var id = Number(u.searchParams.get('id'));
      if (seen.has(id)) return;
      var row = subjectLink.closest('tr') || subjectLink.closest('li, .message, .item, .pm-row, .message-row, .inbox-row');
      if (!row) return;
      var profiles = ownProfileLinks(row);
      var subjectCell = subjectLink.closest('td');
      var profile = profiles.find(function (a) { return !!a.closest('.user, .author, .sender, .recipient'); }) ||
        profiles.find(function (a) { return a.closest('td') !== subjectCell && a.closest('td'); }) || profiles[0];
      var nameCell = $('.user, .author, .sender, .recipient', row);
      if (!nameCell) nameCell = Array.from(row.querySelectorAll('td')).find(function (cell) {
        return cell !== subjectCell && !cell.querySelector('input, time') &&
          !/\d{1,2}:\d{2}/.test(cell.textContent) && normalizeText(cell.textContent);
      });
      var cells = $$('td, time, .date, [class*="date"], [class*="time"]', row).map(function (el) { return normalizeText(el.textContent); });
      var dateText = cells.reverse().find(function (value) { return /\d{1,2}:\d{2}/.test(value); }) || '';
      var entryBox = u.searchParams.get('box') || String(box);
      seen.add(id);
      var badgeText = $$('[title], img[alt]', row).map(function (e) {
        return (e.getAttribute('title') || '') + ' ' + (e.getAttribute('alt') || '');
      }).join(' ');
      var unread = direction(entryBox) === 'in' &&
        (row.matches('.inew, .new-message, .unread') || !!$('.inew, .unread, .new-message', row) ||
         /\u043d\u0435\s*\u043f\u0440\u043e\u0447\u0438\u0442\u0430\u043d|\u043d\u0435\u043f\u0440\u043e\u0447\u0442[\u0435\u0451]\u043d|\u043d\u043e\u0432\u043e\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435|unread/i.test(badgeText));
      var avatar = $('img.avatar, .avatar img, .user-avatar img', row);
      var preview = $('.message-preview, .pm-preview', row);
      result.push({ id: id, box: entryBox, direction: direction(entryBox), href: u.href,
        subject: normalizeText(subjectLink.textContent), partnerId: profile ? linkId(profile) : 0,
        partnerName: profile ? normalizeText(profile.textContent) : normalizeText(nameCell && nameCell.textContent),
        partnerHref: profile ? localUrl(profile.getAttribute('href')).href : '', dateText: dateText,
        timestamp: messageTimestamp(dateText), timePrecise: /\d{1,2}:\d{2}:\d{2}/.test(dateText),
        unread: !!unread, avatar: avatar ? avatar.getAttribute('src') : '',
        preview: preview ? normalizeText(preview.textContent) : '' });
    });

    if (isLoginResponse(doc)) throw new Error('Нужно войти на форум');
    var error = frameError(doc); if (error) throw new Error(error);
    if (messageLinks.length && !result.length) throw new Error('Не распознаны строки списка ЛС. Старые данные сохранены.');
    if (!result.length) {
      var empty = /нет (?:личных )?сообщений|нет писем|сообщений нет|не содержит сообщений|папка пуста|no (?:private )?messages|no messages in|mailbox is empty|folder is empty/i;
      var emptyNodes = $$('.info, .container, .empty, .message-empty, tbody td, p', scope);
      var explicitEmpty = emptyNodes.some(function (node) {
        return !node.closest('#resonance-pm-chat, #resonance-pm-dialogs, #resplash, #maintenance-screen') && empty.test(normalizeText(node.textContent));
      });
      var mailboxTable = $$('table', scope).some(function (table) {
        var head = $('thead', table);
        return head && /отправитель|получатель|sender|recipient/i.test(head.textContent) &&
          /тема|subject/i.test(head.textContent) && !table.querySelector('tbody tr');
      });
      if (!explicitEmpty && !mailboxTable) throw new Error('Не распознана папка ЛС. Сервер вернул неподдерживаемую страницу.');
    }
    return result;
  }
  function maxMailboxPage(doc, box) {
    var max = 1;
    $$('a[href*="messages.php"]', doc).forEach(function (a) {
      var u = localUrl(a.getAttribute('href'));
      if (!u || u.searchParams.has('id') || u.searchParams.get('action')) return;
      if (String(u.searchParams.get('box') || '0') !== String(box)) return;
      var p = Number(u.searchParams.get('p'));
      if (Number.isSafeInteger(p) && p > max) max = p;
    });
    return max;
  }
  async function loadMailbox(box, options) {
    options = options || {};
    var known = new Set(state.mailboxRows.filter(function (e) { return e.box === box; }).map(function (e) { return e.id; }));
    var paging = state.mailboxPaging[box] || { next: 1, max: 1 };
    var p = options.more ? paging.next : 1;
    // A single summary page per folder per action. Older pages are explicit,
    // bounded "Load more" actions, never an automatic crawl.
    if (options.more && p > paging.max) return [];
    try {
      var seed = options.seed;
      var useSeed = seed && !seed.url.searchParams.get('action') && !seed.url.searchParams.has('id') &&
        (seed.url.searchParams.get('box') || '0') === box && Number(seed.url.searchParams.get('p') || 1) === p;
      var doc = useSeed ? seed.doc : await fetchDoc(mailboxUrl(box, p), { fresh: options.fresh, priority: options.more ? -10 : 10 });
      var entries = parseMailboxRows(doc, box); entries.forEach(function (e) { e.page = p; });
      var signature = entries.map(function (e) { return e.id; }).join(',');
      if (options.more && p > 1 && signature && paging.signature === signature && paging.lastPage !== p) {
        throw new Error('\u0424\u043e\u0440\u0443\u043c \u043f\u043e\u0432\u0442\u043e\u0440\u0438\u043b \u0442\u0443 \u0436\u0435 \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u0443');
      }
      var pages = Math.min(10000, maxMailboxPage(doc, box));
      state.mailboxPaging[box] = { next: Math.max(paging.next, p + 1), max: pages, signature: signature, lastPage: p };
      // A burst can insert pages ahead of the previously visited history.
      var newestFirst = entries.every(function (e, i) { return i === 0 || entries[i - 1].id >= e.id; });
      if (p === 1 && known.size && newestFirst && entries.length && !entries.some(function (e) { return known.has(e.id); })) {
        state.mailboxPaging[box].next = 2;
      }
      var snapshot = state.folderSnapshots[box];
      if (p === 1 && (!snapshot || snapshot.signature !== signature || pages === 1)) {
        snapshot = state.folderSnapshots[box] = { next: 1, ids: new Set(), signature: signature, started: Date.now() };
        state.mailboxPaging[box].next = 2;
      }
      if (snapshot && snapshot.next === p) {
        entries.forEach(function (e) { snapshot.ids.add(e.direction + ':' + e.id); });
        snapshot.next = p + 1;
        if (p >= pages) {
          var keepIds = snapshot.ids;
          state.mailboxRows = state.mailboxRows.filter(function (e) {
            if (e.box !== box || keepIds.has(e.direction + ':' + e.id)) return true;
            var key = e.direction + ':' + e.id;
            state.tombstones.add(key); memory.delete(key); return false;
          });
          state.rows = state.rows.filter(function (e) { return !state.tombstones.has(e.direction + ':' + e.id); });
          delete state.folderSnapshots[box];
        }
      }
      entries.forEach(function (e) { state.tombstones.delete(e.direction + ':' + e.id); });
      return entries;
    } catch (error) {
      state.warnings.push('\u041d\u0435 \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u043b\u0430\u0441\u044c \u043f\u0430\u043f\u043a\u0430 ' + box + ', \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u0430 ' + p + '.');
      return [];
    }
  }
  function mailboxBoxes() {
    // Only folders actually exposed by the forum are polled. On this RusFF
    // installation box=2 is not guaranteed to exist; probing a phantom folder
    // was producing a permanent partial-refresh warning.
    var boxes = ['0', '1'];
    $$('a[href*="messages.php"]', document).forEach(function (a) {
      var u = localUrl(a.getAttribute('href'));
      if (!u || u.searchParams.has('id') || u.searchParams.get('action')) return;
      var box = u.searchParams.get('box');
      if (box && /^\d+$/.test(box) && boxes.indexOf(box) < 0) boxes.push(box);
    });
    if (/^\d+$/.test(String(currentBox)) && boxes.indexOf(String(currentBox)) < 0) boxes.push(String(currentBox));
    return boxes;
  }
  async function allMailboxRows(options) {
    options = options || {};
    // A refresh during sending waits for the previous scan, then obtains fresh
    // rows; it cannot mistake a scan started before the POST for its result.
    while (state.mailboxJob) await state.mailboxJob;
    state.mailboxJob = (async function () {
      state.warnings = [];
      var boxes = options.boxes || mailboxBoxes();
      var incremental = !!options.incremental && state.mailboxReady;
      var lists = await mapLimit(boxes, 3, function (box) {
        return loadMailbox(box, { incremental: incremental, seed: options.seed, more: options.more, fresh: options.afterSend || options.fresh });
      });
      var map = new Map();
      state.mailboxRows.forEach(function (e) { map.set(e.direction + ':' + e.id, e); });
      lists.flat().forEach(function (e) {
        map.set(e.direction + ':' + e.id, e);
      });
      state.mailboxRows = Array.from(map.values());
      state.mailboxReady = true;
      scheduleCacheSave();
      return state.mailboxRows;
    })();
    try { return await state.mailboxJob; } finally { state.mailboxJob = null; }
  }
  function hasOlderPages() {
    if (state.chat && apiState.history) return !apiState.historyDone;
    if (state.list && apiState.recent) return !apiState.recentDone;
    return Object.keys(state.mailboxPaging).some(function (box) {
      var page = state.mailboxPaging[box]; return page && page.next <= page.max;
    });
  }
  function historyEntries(all, current) {
    var entries = all.filter(function (e) { return matchesPartner(e, state.partner); });
    if (current && !apiState.history && !state.tombstones.has(current.direction + ':' + current.id) && !entries.some(function (e) { return e.id === current.id && e.direction === current.direction; })) entries.push(current);
    entries.sort(compareMessages);
    return entries;
  }
  function historyStatus() {
    var node = state.chat ? $('.rpmc-history-status', state.chat) : state.list && $('.rpmc-list-status', state.list);
    if (!node) return;
    var text = state.historyError || (state.historySync ? '\u041f\u043e\u0434\u0433\u0440\u0443\u0436\u0430\u044e \u0438\u0441\u0442\u043e\u0440\u0438\u044e\u2026' : '');
    if (node.textContent !== text) node.textContent = text;
  }
  function mergeVisible(messages, scrollToEnd) {
    var rows = new Map(state.rows.map(function (e) { return [e.direction + ':' + e.id, e]; }));
    messages.forEach(function (e) { rows.set(e.direction + ':' + e.id, e); });
    state.rows = coalesceDeliveryCopies(Array.from(rows.values()));
    if (state.chat) renderMessages(state.rows, scrollToEnd);
  }
  function showCachedHistory(scrollToEnd) {
    var cached = historyEntries(state.mailboxRows, state.current).filter(function (e) {
      return memory.has(e.direction + ':' + e.id);
    }).slice(-30).map(function (e) { return Object.assign({}, memory.get(e.direction + ':' + e.id), e, { html: memory.get(e.direction + ':' + e.id).html, date: e.dateText || memory.get(e.direction + ':' + e.id).date, timestamp: e.timestamp || memory.get(e.direction + ':' + e.id).timestamp }); });
    mergeVisible(cached, scrollToEnd);
  }
  function canSyncHistory() {
    return state.autoHistory && !state.stopped && !document.hidden && !state.submitting &&
      networkPause() <= Date.now() && !(typeof navigator !== 'undefined' && navigator.onLine === false);
  }
  async function syncHistory() {
    if (!canSyncHistory() || state.historySync || state.refresh || state.historyBusy) return;
    state.historySync = true;
    try { await loadOlder(); }
    finally { state.historySync = null; historyStatus(); }
  }

  function startHistorySync() {
    if (!canSyncHistory()) return;
    return syncHistory().catch(function (error) { state.historyError = error.message; historyStatus(); });
  }


  function addHistoryControls(root) {
    var bar = document.createElement('div'); bar.className = 'rpmc-history-controls';
    bar.innerHTML = '<button type="button" data-more>\u0417\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u0441\u0442\u0430\u0440\u044b\u0435</button> <button type="button" data-latest>\u041a \u043f\u043e\u0441\u043b\u0435\u0434\u043d\u0438\u043c</button> <button type="button" data-reindex>\u0421\u0432\u0435\u0440\u0438\u0442\u044c \u0438\u0441\u0442\u043e\u0440\u0438\u044e</button>';
    var target = $('.rpmc-messages, .rpmc-dialog-items', root); target.before(bar);
    $('[data-more]', bar).addEventListener('click', function () { loadOlder(); });
    $('[data-latest]', bar).addEventListener('click', function () {
      state.displayBefore = null; state.displayCount = 30;
      if (state.chat) refreshHistory(true, { fresh: true }); else refreshDialogues(false, null, { fresh: true });
    });
    $('[data-reindex]', bar).addEventListener('click', async function () {
      if (state.historyBusy || state.submitting || state.refresh) return;
      state.mailboxPaging = {}; state.folderSnapshots = {};
      apiState.historySkip = 0; apiState.recentSkip = 0; apiState.historyDone = false; apiState.recentDone = false;
      if (state.chat) await refreshHistory(false, { fresh: true }); else await refreshDialogues(false, null, { fresh: true });
      state.historyError = state.warnings.length ? '\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043d\u0430\u0447\u0430\u0442\u044c \u0441\u0432\u0435\u0440\u043a\u0443. \u041f\u043e\u0432\u0442\u043e\u0440\u0438 \u043f\u043e\u0437\u0436\u0435.' : hasOlderPages()
        ? '\u0421\u0432\u0435\u0440\u043a\u0430 \u043d\u0430\u0447\u0430\u0442\u0430. \u00ab\u0417\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u0441\u0442\u0430\u0440\u044b\u0435\u00bb \u043f\u0440\u043e\u0434\u043e\u043b\u0436\u0430\u0435\u0442 \u0435\u0435 \u043f\u043e\u0440\u0446\u0438\u044f\u043c\u0438. \u0423\u0434\u0430\u043b\u0435\u043d\u0438\u044f \u043f\u043e\u0434\u0442\u0432\u0435\u0440\u0436\u0434\u0430\u044e\u0442\u0441\u044f \u043f\u043e\u0441\u043b\u0435 \u043e\u0431\u0445\u043e\u0434\u0430 \u0432\u0441\u0435\u0439 \u043f\u0430\u043f\u043a\u0438.' : '\u0418\u0441\u0442\u043e\u0440\u0438\u044f \u0441\u0432\u0435\u0440\u0435\u043d\u0430.';
      historyStatus();
    });
  }
  async function loadOlder() {
    if (state.historyBusy || state.submitting || state.refresh || state.mailboxJob) return;
    state.historyBusy = true; state.historyError = '\u0417\u0430\u0433\u0440\u0443\u0436\u0430\u044e \u0441\u0442\u0430\u0440\u044b\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f\u2026'; historyStatus();
    try {
      if (state.chat && apiState.history) {
        state.displayCount = Math.min(DISPLAY_LIMIT, state.displayCount + 30);
        await apiHistory(true, false); state.historyError = apiState.historyDone ? 'Вся история загружена.' : 'Загружена следующая часть истории.'; return;
      }
      if (state.list && apiState.recent) { await apiRecent(true); renderDialogues(); state.historyError = apiState.recentDone ? 'Все диалоги загружены.' : 'Загружена следующая часть диалогов.'; return; }
      var entries = state.chat ? historyEntries(state.mailboxRows, state.current) : [];
      var earliest = state.rows.length ? state.rows[0] : null;
      var missing = entries.filter(function (e) { return !state.rows.some(function (r) { return r.direction === e.direction && r.id === e.id; }); });
      if (!missing.length && hasOlderPages()) {
        var boxes = Object.keys(state.mailboxPaging).filter(function (box) { var p = state.mailboxPaging[box]; return p.next <= p.max; });
        var box = boxes[state.olderRound++ % boxes.length] || boxes[0];
        await allMailboxRows({ more: true, fresh: true, boxes: [box] });
        entries = state.chat ? historyEntries(state.mailboxRows, state.current) : [];
        missing = entries.filter(function (e) { return !state.rows.some(function (r) { return r.direction === e.direction && r.id === e.id; }); });
      }
      if (state.chat) {
        var oldLength = state.rows.length;
        var loaded = await mapLimit(missing.slice(-20).reverse(), 1, function (e) { return loadMessage(e, { priority: -10 }); });
        state.displayCount = Math.min(DISPLAY_LIMIT, state.displayCount + 20);
        mergeVisible(loaded, false);
        if (state.rows.length > DISPLAY_LIMIT || state.displayBefore != null) {
          var newCount = state.rows.length - oldLength;
          state.displayBefore = state.displayBefore == null ? Math.max(DISPLAY_LIMIT, state.rows.length - 20) : Math.max(DISPLAY_LIMIT, state.displayBefore + newCount - 20);
          renderMessages(state.rows, false);
        }
      } else renderDialogues();
      state.historyError = state.warnings.length ? '\u041d\u0435 \u0432\u0441\u0435 \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u044b \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u043b\u0438\u0441\u044c. \u041f\u043e\u0432\u0442\u043e\u0440\u0438 \u00ab\u0417\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u0441\u0442\u0430\u0440\u044b\u0435\u00bb.' : hasOlderPages()
        ? '\u0417\u0430\u0433\u0440\u0443\u0436\u0435\u043d\u0430 \u043e\u0447\u0435\u0440\u0435\u0434\u043d\u0430\u044f \u0447\u0430\u0441\u0442\u044c. \u0414\u043b\u044f \u043f\u0440\u043e\u0434\u043e\u043b\u0436\u0435\u043d\u0438\u044f \u043d\u0430\u0436\u043c\u0438 \u00ab\u0417\u0430\u0433\u0440\u0443\u0437\u0438\u0442\u044c \u0441\u0442\u0430\u0440\u044b\u0435\u00bb.' : '\u0412\u0441\u0435 \u0438\u0437\u0432\u0435\u0441\u0442\u043d\u044b\u0435 \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u044b \u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440\u0435\u043d\u044b.';
    } catch (error) { state.historyError = error.message; }
    finally { state.historyBusy = false; historyStatus(); scheduleCacheSave(); }
  }

  function nativePost(doc) {
    return $('#pun-messages .post', doc) || $('#pun-main .post', doc) || $('.post', doc);
  }
  function replyHref(doc) {
    var post = nativePost(doc), scope = post || doc;
    var links = $$('a[href*="messages.php"]', scope);
    if (post) links = links.concat($$('.pl-reply a, a[href*="messages.php"]', doc));
    var link = links.find(function (a) {
      var u = localUrl(a.getAttribute('href'));
      if (!u || !/\/messages\.php$/i.test(u.pathname)) return false;
      if (/delete|remove|clear/i.test(u.searchParams.get('action') || '')) return false;
      return /^(\u043e\u0442\u0432\u0435\u0442\u0438\u0442\u044c|reply)$/i.test(normalizeText(a.textContent)) || !!a.closest('.pl-reply');
    });
    return link ? localUrl(link.getAttribute('href')).href : '';
  }
  function currentRowFromPage() {
    var post = nativePost(document); if (!post) return null;
    var profile = ownProfileLinks($('.post-author', post) || post)[0];
    var subject = $('#pun-main h1') || $('#pun-messages h1');
    return { id: currentMessageId, box: currentBox, direction: direction(currentBox), href: location.href,
      subject: normalizeText(subject && subject.textContent), partnerId: profile ? linkId(profile) : 0,
      partnerName: normalizeText(profile && profile.textContent), partnerHref: profile ? localUrl(profile.getAttribute('href')).href : '', dateText: '' };
  }
  function matchesPartner(entry, partner) {
    if (entry.partnerId && partner.id) return entry.partnerId === partner.id;
    return !!entry.partnerName && normalizeText(entry.partnerName).toLowerCase() === normalizeText(partner.name).toLowerCase();
  }
  function cleanSubject(value) {
    return normalizeText(value).replace(/^(?:(?:re|fw|fwd)(?:\(\d+\))?\s*:\s*)+/i, '').trim();
  }
  function sourceIds(entry) { return entry.sourceIds || [entry.id]; }
  var messagePartsCache = new Map();
  function messageParts(message) {
    var html = message.html || '';
    if (messagePartsCache.has(html)) return messagePartsCache.get(html);
    var node = document.createElement('div'); node.innerHTML = html;
    var media = $$('img[src], audio[src], video[src], source[src], iframe[src]', node).map(function (element) {
      var value = element.getAttribute('src'), link = element.closest('a[href]');
      if (element.tagName === 'IMG' && link && /\.(?:gif|png|jpe?g|webp)(?:[?#]|$)/i.test(link.getAttribute('href'))) value = link.getAttribute('href');
      var url = urlOf(value); return url ? url.href : value;
    });
    var links = $$('a[href]', node).filter(function (a) {
      return !($('img', a) && /\.(?:gif|png|jpe?g|webp)(?:[?#]|$)/i.test(a.getAttribute('href')));
    }).map(function (a) { var u = urlOf(a.getAttribute('href')); return u ? u.href : ''; });
    $$('.rpmc-quote-summary, img, audio, video, iframe, .post-sig', node).forEach(function (el) { el.remove(); });
    $$('br', node).forEach(function (el) { el.replaceWith(document.createTextNode(' ')); });
    $$('p, div, li', node).forEach(function (el) { el.appendChild(document.createTextNode(' ')); });
    var text = normalizeText(node.textContent);
    var parts = { text: text, media: media, signature: JSON.stringify([text, media, links]) };
    messagePartsCache.set(html, parts);
    if (messagePartsCache.size > 2000) messagePartsCache.delete(messagePartsCache.keys().next().value);
    return parts;
  }
  function mediaInMessage(message) { return messageParts(message).media; }
  function mediaInDraft(text) {
    var result = [], pattern = /\[(?:img|audio|video)(?:=[^\]]*)?\]([\s\S]*?)\[\/(?:img|audio|video)\]/gi, match;
    while ((match = pattern.exec(String(text || '')))) {
      var value = match[1].trim(), url = urlOf(value); result.push(url ? url.href : value);
    }
    return result;
  }
  function deliverySignature(message) { return messageParts(message).signature; }
  function deliveryTime(message) {
    // Mailbox timestamps are refreshed across midnight; cached post headings
    // can still say "Today" for a letter sent yesterday.
    var text = normalizeText(message.dateText || message.date).replace(/\s*#\d+\s*$/, '');
    var clock = text.match(/\d{1,2}:\d{2}:\d{2}/); if (!clock) return '';
    var day = text.replace(clock[0], '').trim(), now = new Date();
    if (/^(\u0441\u0435\u0433\u043e\u0434\u043d\u044f|today)$/i.test(day) || /^(\u0432\u0447\u0435\u0440\u0430|yesterday)$/i.test(day)) {
      if (/^(\u0432\u0447\u0435\u0440\u0430|yesterday)$/i.test(day)) now.setDate(now.getDate() - 1);
      day = [now.getFullYear(), now.getMonth() + 1, now.getDate()].join('-');
    }
    return day + ' ' + clock[0];
  }
  function coalesceDeliveryCopies(messages) {
    var byId = new Map();
    messages.forEach(function (e) { if (!state.tombstones.has(e.direction + ':' + e.id)) byId.set(e.direction + ':' + e.id, e); });
    return Array.from(byId.values()).sort(compareMessages);
  }

  function normalizeQuotes(root) {
    function convert(box) {
      if (!box.parentNode) return;
      var body = Array.from(box.children).find(function (el) { return el.tagName === 'BLOCKQUOTE'; }) || box;
      var cite = $('cite', body) || Array.from(body.children).find(function (el) { return /^H[3-6]$/.test(el.tagName); });
      if (!cite && body !== box) cite = Array.from(box.children).find(function (el) { return el.tagName === 'CITE' || /^H[3-6]$/.test(el.tagName); });
      var author = cite ? normalizeText(cite.textContent).replace(/\s+(?:\u043d\u0430\u043f\u0438\u0441\u0430\u043b\(\u0430\)|\u043d\u0430\u043f\u0438\u0441\u0430\u043b|\u043d\u0430\u043f\u0438\u0441\u0430\u043b\u0430|wrote)\s*:?.*$/i, '').replace(/^\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435\s+\u043e\u0442\s+/i, '').replace(/:\s*$/, '') : '';
      if (cite) {
        var p = cite.parentElement; cite.remove();
        if (p && p !== body && p.tagName === 'P' && !normalizeText(p.textContent) && !p.children.length) p.remove();
      }
      var details = root.ownerDocument.createElement('details'); details.className = 'rpmc-quote';
      var summary = root.ownerDocument.createElement('summary'); summary.className = 'rpmc-quote-summary';
      var label = root.ownerDocument.createElement('span'); label.className = 'rpmc-quote-author'; label.textContent = author || '\u0426\u0438\u0442\u0430\u0442\u0430';
      var preview = root.ownerDocument.createElement('span'); preview.className = 'rpmc-quote-preview';
      var excerpt = body.cloneNode(true);
      $$('.rpmc-quote, .quote-box, blockquote, .quote', excerpt).forEach(function (el) { el.remove(); });
      preview.textContent = normalizeText(excerpt.textContent).slice(0, 220) || '\u041f\u043e\u043a\u0430\u0437\u0430\u0442\u044c \u0446\u0438\u0442\u0430\u0442\u0443';
      summary.append(label, preview);
      var content = root.ownerDocument.createElement('div'); content.className = 'rpmc-quote-content';
      while (body.firstChild) content.appendChild(body.firstChild);
      details.append(summary, content); box.replaceWith(details);
    }
    $$('.quote-box, div.quote', root).reverse().forEach(convert);
    // A wrapper's direct blockquote was unwrapped above; remaining ones are standalone quotes.
    $$('blockquote', root).reverse().forEach(convert);
  }
  function normalizeSpoilers(root) {
    // Imported forum HTML loses its inline click handlers. Native details work
    // without executing message scripts and support keyboard/nested spoilers.
    $$('.spoiler-box, div.spoiler, .spoiler-block', root).reverse().forEach(function (box) {
      if (!box.parentNode || box.tagName === 'DETAILS') return;
      var children = Array.from(box.children);
      var body = children.find(function (el) {
        return el.matches('blockquote, .spoiler-content, .spoiler-body, .spoiler-text');
      });
      if (!body) return;
      var title = children.find(function (el) {
        return el !== body && el.matches('.spoiler-title, .spoiler-header, .spoiler-toggle, summary, h3, h4, button, div');
      });
      var details = root.ownerDocument.createElement('details'); details.className = 'rpmc-spoiler';
      var summary = root.ownerDocument.createElement('summary');
      summary.textContent = normalizeText(title && title.textContent) || '\u0421\u043f\u043e\u0439\u043b\u0435\u0440';
      var content = root.ownerDocument.createElement('div'); content.className = 'rpmc-spoiler-content';
      while (body.firstChild) content.appendChild(body.firstChild);
      details.append(summary, content); box.replaceWith(details);
    });
  }
  function safeMessageHtml(content) {
    var clone = content.cloneNode(true);
    // Forum lazy spoilers are inert templates; expand only in the known wrapper.
    $$('.spoiler-box script[type="text/html"], .spoiler-content script[type="text/html"]', clone).forEach(function (el) {
      var parsed = new DOMParser().parseFromString(el.textContent, 'text/html');
      var body = clone.ownerDocument.createElement('div'); body.className = 'spoiler-content';
      body.innerHTML = messagePurifier.sanitize(parsed.body.innerHTML); el.replaceWith(body);
    });
    $$('.post-sig, .post-links, .post-rating', clone).forEach(function (el) { el.remove(); });
    var options = {
      ALLOWED_TAGS: ['p','div','span','br','hr','b','strong','i','em','u','s','strike','del','sup','sub','a','img','blockquote','cite','pre','code','ul','ol','li','table','thead','tbody','tfoot','tr','td','th','caption','h3','h4','h5','h6','details','summary','audio','video','source','iframe'],
      ALLOWED_ATTR: ['href','src','alt','title','class','style','colspan','rowspan','controls','poster','type','open','start'],
      ALLOW_DATA_ATTR: false, ALLOW_ARIA_ATTR: false, RETURN_DOM_FRAGMENT: true
    };
    var fragment = messagePurifier.sanitize(clone.innerHTML, options);
    clone.replaceChildren(fragment);
    var classes = /^(?:quote-box|quote|spoiler-box|spoiler|spoiler-block|spoiler-content|spoiler-body|spoiler-text|spoiler-title|spoiler-header|spoiler-toggle|postimg|rpmc-(?:quote|quote-summary|quote-author|quote-preview|quote-content|spoiler|spoiler-content))$/;
    $$('*', clone).forEach(function (el) {
      if (el.hasAttribute('class')) el.className = el.className.split(/\s+/).filter(function (v) { return classes.test(v); }).join(' ');
      var declarations = [];
      ['color','background-color','font-family','font-size','font-weight','font-style','text-align','text-decoration','white-space'].forEach(function (prop) {
        var value = el.style.getPropertyValue(prop);
        if (value && !/url\s*\(|expression|var\s*\(|[<>\\]/i.test(value) && value.length < 180) declarations.push([prop, value]);
      });
      el.removeAttribute('style'); declarations.forEach(function (pair) { el.style.setProperty(pair[0], pair[1]); });
      ['href','src','poster'].forEach(function (attr) {
        if (!el.hasAttribute(attr)) return;
        var u = urlOf(el.getAttribute(attr));
        if (!u || !(attr === 'href' ? /^(https?:|mailto:)$/ : /^https?:$/).test(u.protocol)) el.removeAttribute(attr);
        else el.setAttribute(attr, u.href);
      });
      if (el.tagName === 'A') el.setAttribute('rel', 'noopener noreferrer');
      if (el.tagName === 'IFRAME') {
        var url = urlOf(el.getAttribute('src') || '');
        if (!url || url.protocol !== 'https:' || !/^(?:www\.)?(?:youtube\.com|youtube-nocookie\.com|player\.vimeo\.com)$/.test(url.hostname) ||
            !(url.hostname === 'player.vimeo.com' ? /^\/video\// : /^\/embed\//).test(url.pathname)) { el.remove(); return; }
        el.setAttribute('sandbox', 'allow-scripts allow-presentation'); el.setAttribute('loading', 'lazy'); el.setAttribute('referrerpolicy', 'no-referrer');
      }
      if (el.tagName === 'IMG') { el.setAttribute('loading', 'lazy'); el.setAttribute('decoding', 'async'); }
      if (/^(AUDIO|VIDEO)$/.test(el.tagName)) { el.setAttribute('controls', ''); el.setAttribute('preload', 'none'); }
    });
    normalizeSpoilers(clone); normalizeQuotes(clone);
    return clone.innerHTML;
  }

  function extractMessage(doc) {
    var post = nativePost(doc);
    var body = post && ($('.post-content', post) || $('.post-body', post));
    if (!body) throw new Error('\u041d\u0435 \u043d\u0430\u0439\u0434\u0435\u043d \u0442\u0435\u043a\u0441\u0442 \u043f\u0438\u0441\u044c\u043c\u0430');
    var date = $('h3', post);
    return { html: safeMessageHtml(body), timestamp: messageTimestamp(normalizeText(date && date.textContent)), date: normalizeText(date && date.textContent).replace(/^\u041f\u043e\u0434\u0435\u043b\u0438\u0442\u044c\u0441\u044f\s*\d*/i, '').trim(), reply: replyHref(doc) };
  }
  async function loadMessage(entry, options) {
    var key = entry.direction + ':' + entry.id;
    if (!memory.has(key)) {
      var doc = entry.id === currentMessageId && entry.box === currentBox ? document : await fetchDoc(entry.href, options);
      memory.set(key, extractMessage(doc));
      scheduleCacheSave();
    }
    var cached = memory.get(key);
    return Object.assign({}, cached, entry, { html: cached.html, date: entry.dateText || cached.date, timestamp: entry.timestamp || cached.timestamp, reply: cached.reply });
  }
  function avatarHtml(partner, cls) {
    var cached = readAvatar(partner.id), src = partner.avatar || (cached && cached.src);
    var u = urlOf(src);
    return '<div class="' + cls + '" data-rpmc-avatar="' + esc(partner.id || '') + '" data-avatar-profile="' +
      esc(partner.href || '') + '" data-avatar-letter="' + esc((partner.name || '?').charAt(0).toUpperCase()) + '">' +
      (src && u && /^https?:$/.test(u.protocol)
      ? '<img src="' + esc(u.href) + '" alt="">'
      : '<span class="rpmc-avatar-placeholder">' + esc((partner.name || '?').charAt(0).toUpperCase()) + '</span>') + '</div>';
  }
  function readAvatar(id) {
    if (!id) return null;
    if (!avatars.has(id)) {
      try {
        var cached = JSON.parse(window.sessionStorage.getItem('rpmc-avatar:' + id) || 'null');
        if (cached && typeof cached.src === 'string' && cached.expires > Date.now()) avatars.set(id, cached);
      } catch (_) {}
    }
    var value = avatars.get(id);
    return value && value.expires > Date.now() ? value : null;
  }
  function storeAvatar(id, src, ttl) {
    if (!id) return;
    var value = { src: src || '', expires: Date.now() + (ttl || 600000) };
    avatars.set(id, value);
    try { window.sessionStorage.setItem('rpmc-avatar:' + id, JSON.stringify(value)); } catch (_) {}
  }
  function profileAvatar(doc, href) {
    var scope = $('#viewprofile, #profile-left, #profile1, #profile', doc);
    var avatar = scope && $('#pa-avatar img, #profile-avatar img, .pa-avatar img, .profile-avatar img, img.avatar', scope);
    if (!avatar) avatar = $('#pa-avatar img, #profile-avatar img', doc);
    if (!avatar) return '';
    var u = urlOf(avatar.getAttribute('data-src') || avatar.getAttribute('src'), href);
    return u && /^https?:$/.test(u.protocol) ? u.href : '';
  }
  function paintAvatar(id, src) {
    var valid = src && urlOf(src);
    if (!valid || !/^https?:$/.test(valid.protocol)) return;
    src = valid.href;
    $$('[data-rpmc-avatar]').forEach(function (holder) {
      if (String(id) !== holder.getAttribute('data-rpmc-avatar')) return;
      var current = $('img', holder);
      if (current && current.getAttribute('src') === src) return;
      var img = document.createElement('img'); img.alt = ''; img.src = src; img.decoding = 'async';
      img.addEventListener('error', function () {
        if (!img.parentNode) return;
        storeAvatar(id, '', 60000);
        var fallback = document.createElement('span'); fallback.className = 'rpmc-avatar-placeholder';
        fallback.textContent = holder.getAttribute('data-avatar-letter') || '?'; img.replaceWith(fallback);
      }, { once: true });
      holder.replaceChildren(img);
    });
  }
  async function runAvatarQueue() {
    if (state.avatarWorkers >= 2) return;
    state.avatarWorkers++;
    try {
      while (state.avatarQueue.length) {
        var job = state.avatarQueue.shift();
        try {
          var cached = readAvatar(job.id);
          var src = cached ? cached.src : profileAvatar(await fetchDoc(job.href), job.href);
          if (!cached) storeAvatar(job.id, src);
          paintAvatar(job.id, src); job.resolve(src);
        } catch (_) { storeAvatar(job.id, '', 60000); job.resolve(''); }
        finally { avatarJobs.delete(job.id); }
      }
    } finally { state.avatarWorkers--; }
  }
  function loadAvatar(id, href) {
    var u = localUrl(href);
    if (!id || !u || !/\/profile\.php$/i.test(u.pathname) || Number(u.searchParams.get('id')) !== id) return Promise.resolve('');
    var cached = readAvatar(id);
    if (cached) { paintAvatar(id, cached.src); return Promise.resolve(cached.src); }
    if (avatarJobs.has(id)) return avatarJobs.get(id);
    var resolveJob, result = new Promise(function (resolve) { resolveJob = resolve; });
    avatarJobs.set(id, result); state.avatarQueue.push({ id: id, href: u.href, resolve: resolveJob });
    runAvatarQueue(); return result;
  }
  function hydrateAvatars(root) {
    if (!root || state.initializing) return;
    if (!state.avatarObserver && typeof IntersectionObserver !== 'undefined') {
      state.avatarObserver = new IntersectionObserver(function (entries) {
        entries.forEach(function (entry) {
          if (!entry.isIntersecting) return;
          var node = entry.target; state.avatarObserver.unobserve(node); node._rpmcObserving = false;
          loadAvatar(Number(node.getAttribute('data-rpmc-avatar')), node.getAttribute('data-avatar-profile'));
        });
      }, { rootMargin: '120px' });
    }
    var painted = new Set();
    $$('[data-rpmc-avatar]', root).forEach(function (holder, index) {
      var id = Number(holder.getAttribute('data-rpmc-avatar'));
      if (!id) return;
      var known = readAvatar(id);
      if (known) {
        if (!painted.has(id)) { painted.add(id); paintAvatar(id, known.src); }
        return;
      }
      if ($('img', holder)) {
        storeAvatar(id, $('img', holder).getAttribute('src')); return;
      }
      if (state.avatarObserver) {
        if (!holder._rpmcObserving) { holder._rpmcObserving = true; state.avatarObserver.observe(holder); }
      } else if (index < 30) loadAvatar(id, holder.getAttribute('data-avatar-profile'));
    });
  }
  function wordMessages(count) {
    if (count % 10 === 1 && count % 100 !== 11) return '\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435';
    if (count % 10 >= 2 && count % 10 <= 4 && (count % 100 < 12 || count % 100 > 14)) return '\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f';
    return '\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0439';
  }
  function setStatus(text, error) {
    var node = state.chat && $('.rpmc-status', state.chat); if (!node) return;
    node.textContent = text; node.classList.toggle('error', !!error);
    node.classList.toggle('is-sending', !!text && !error && state.submitting && state.pending && state.pending.kind === 'send');
  }
  function rememberHidden(el) {
    if (!el || el.contains(state.chat) || el.contains(state.list) || el.closest('#resonance-pm-modes') ||
        state.hidden.some(function (item) { return item.node === el; })) return;
    state.hidden.push({ node: el, value: el.style.getPropertyValue('display'), priority: el.style.getPropertyPriority('display') });
    el.style.setProperty('display', 'none', 'important');
  }
  function hideMailboxNodes() {
    var root = $('#pun-main') || $('#pun-messages'); if (!root) return;
    $$('table', root).forEach(function (table) {
      if (table.closest('#resonance-pm-chat, #resonance-pm-dialogs')) return;
      var isMailbox = $$('a[href*="messages.php"]', table).some(function (a) {
        var u = localUrl(a.getAttribute('href')); return u && Number(u.searchParams.get('id')) > 0;
      });
      if (!isMailbox) isMailbox = /\u043e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u0435\u043b\u044c|\u043f\u043e\u043b\u0443\u0447\u0430\u0442\u0435\u043b\u044c|sender|recipient/i.test(normalizeText(table.querySelector('thead') && table.querySelector('thead').textContent));
      if (!isMailbox) return;
      var form = table.closest('form');
      rememberHidden(form && !form.querySelector('textarea') ? form : table);
    });
    $$('.linkst, .linksb, .pagelink, .pagelinks', root).forEach(function (el) {
      if (!el.closest('#resonance-pm-chat, #resonance-pm-dialogs')) rememberHidden(el);
    });
  }
  function mountModes() {
    var root = $('#pun-main') || $('#pun-messages'); if (!root) return;
    var nav = document.createElement('nav'); nav.id = 'resonance-pm-modes';
    nav.setAttribute('aria-label', '\u0420\u0435\u0436\u0438\u043c \u043b\u0438\u0447\u043d\u044b\u0445 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0439');
    var chatTarget = currentMessageId && !pageUrl.searchParams.get('action') ? location.href : mailboxUrl('0', 1);
    nav.innerHTML = '<a href="' + esc(modeLink(chatTarget, 'chat')) + '" data-pm-mode="chat">\u0414\u0438\u0430\u043b\u043e\u0433\u0438</a>' +
      '<a href="' + esc(modeLink(location.href, 'native')) + '" data-pm-mode="native">\u041e\u0431\u044b\u0447\u043d\u044b\u0435 \u041b\u0421</a>';
    $$('[data-pm-mode]', nav).forEach(function (a) {
      if (a.dataset.pmMode === state.mode) { a.classList.add('is-active'); a.setAttribute('aria-current', 'page'); }
      a.addEventListener('click', function () { setMode(a.dataset.pmMode); });
    });
    root.insertBefore(nav, root.firstChild);
  }
  function groupConversations(rows) {
    var names = new Map();
    rows.forEach(function (e) {
      if (!e.partnerId || !e.partnerName) return;
      var name = normalizeText(e.partnerName).toLowerCase();
      if (!names.has(name)) names.set(name, new Set());
      names.get(name).add(e.partnerId);
    });
    var groups = new Map();
    rows.forEach(function (e) {
      var name = normalizeText(e.partnerName).toLowerCase(), ids = names.get(name);
      var id = e.partnerId || (ids && ids.size === 1 ? Array.from(ids)[0] : 0);
      // A mailbox/navigation link without a companion is not a dialogue row.
      // Older broad parsing could cache such a row and render it as “Удалённый пользователь”.
      if (!id && !name) return;
      var key = id ? 'id:' + id : name && !(ids && ids.size > 1) ? 'name:' + name : 'message:' + e.id;
      if (!groups.has(key)) groups.set(key, { key: key, id: id, name: e.partnerName || '\u0423\u0434\u0430\u043b\u0451\u043d\u043d\u044b\u0439 \u043f\u043e\u043b\u044c\u0437\u043e\u0432\u0430\u0442\u0435\u043b\u044c',
        href: e.partnerHref || '', avatar: e.avatar || '', latest: e, unread: 0 });
      var group = groups.get(key);
      if (compareMessages(e, group.latest) > 0) group.latest = e;
      if (e.avatar && !group.avatar) group.avatar = e.avatar;
      if (e.direction === 'in' && e.unread && !state.readIds.has(e.id)) group.unread += e.unreadCount == null ? 1 : e.unreadCount;
    });
    return Array.from(groups.values()).sort(function (a, b) { return compareMessages(b.latest, a.latest); });
  }
  function mountDialogues() {
    var root = $('#pun-main') || $('#pun-messages'); if (!root) throw new Error('\u041d\u0435 \u043d\u0430\u0439\u0434\u0435\u043d \u0441\u043f\u0438\u0441\u043e\u043a \u041b\u0421');
    var list = document.createElement('section'); list.id = 'resonance-pm-dialogs';
    var compose = $$('a[href*="messages.php"]').find(function (a) {
      var u = localUrl(a.getAttribute('href'));
      return u && /^(new|send)$/i.test(u.searchParams.get('action') || '') &&
        /\u043d\u043e\u0432\u043e\u0435|\u043d\u0430\u043f\u0438\u0441\u0430\u0442\u044c|\u043e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c|new message/i.test(a.textContent);
    });
    list.innerHTML = '<header class="rpmc-dialog-head"><strong>\u0414\u0438\u0430\u043b\u043e\u0433\u0438</strong>' +
      (compose ? '<a class="rpmc-new-message" href="' + esc(nativeUrl(compose.getAttribute('href'))) + '">\u041d\u043e\u0432\u043e\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435</a>' : '') +
      '<button type="button" class="rpmc-refresh" aria-label="\u041e\u0431\u043d\u043e\u0432\u0438\u0442\u044c \u0434\u0438\u0430\u043b\u043e\u0433\u0438">\u21bb</button></header>' +
      '<div class="rpmc-dialog-controls"><input type="search" class="rpmc-dialog-filter" placeholder="\u041d\u0430\u0439\u0442\u0438 \u0441\u043e\u0431\u0435\u0441\u0435\u0434\u043d\u0438\u043a\u0430" aria-label="\u041d\u0430\u0439\u0442\u0438 \u0441\u043e\u0431\u0435\u0441\u0435\u0434\u043d\u0438\u043a\u0430">' +
      '<label><input type="checkbox" class="rpmc-unread-filter"> \u041d\u0435\u043f\u0440\u043e\u0447\u0438\u0442\u0430\u043d\u043d\u044b\u0435</label></div>' +
      '<div class="rpmc-list-status" role="status"></div><div class="rpmc-dialog-items"></div>';
    var modes = $('#resonance-pm-modes');
    if (modes && modes.parentNode === root) root.insertBefore(list, modes.nextSibling);
    else root.insertBefore(list, root.firstChild);
    state.list = list; addHistoryControls(list);
    $('.rpmc-dialog-filter', list).addEventListener('input', function (e) {
      state.listFilter = normalizeText(e.target.value).toLowerCase(); renderDialogues();
    });
    $('.rpmc-unread-filter', list).addEventListener('change', function (e) {
      state.unreadOnly = e.target.checked; renderDialogues();
    });
    $('.rpmc-refresh', list).addEventListener('click', function () { refreshDialogues(false); });
  }
  function renderDialogues() {
    if (!state.list) return;
    var groups = groupConversations(apiState.recent ? apiState.dialogRows : state.mailboxRows).filter(function (g) {
      return (!state.unreadOnly || g.unread) && (!state.listFilter || g.name.toLowerCase().includes(state.listFilter));
    });
    var area = $('.rpmc-dialog-items', state.list), previous = new Map();
    Array.from(area.children).forEach(function (node) { if (node.dataset.partner) previous.set(node.dataset.partner, node); });
    var retained = new Set();
    groups.forEach(function (group, i) {
      var last = group.latest, node = previous.get(group.key);
      var preview = last.preview || (last.subject ? '\u041f\u043e\u0441\u043b\u0435\u0434\u043d\u0435\u0435 \u043f\u0438\u0441\u044c\u043c\u043e: ' + cleanSubject(last.subject) : '\u041e\u0442\u043a\u0440\u044b\u0442\u044c \u043f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0443');
      var html = avatarHtml(group, 'rpmc-dialog-avatar') +
        '<span class="rpmc-dialog-body"><strong>' + esc(group.name) + '</strong><span class="rpmc-dialog-preview">' +
        esc((last.direction === 'out' ? '\u0412\u044b \xb7 ' : '') + preview) + '</span></span><span class="rpmc-dialog-meta"><time>' +
        esc(last.dateText) + '</time>' + (group.unread ? '<span class="rpmc-unread-badge">' + group.unread + '</span>' : '') + '</span>';
      if (!node) { node = document.createElement('a'); node.className = 'rpmc-dialog'; node.dataset.partner = group.key; }
      node.href = chatUrl(last.href); if (last.page) { var route = new URL(node.href); route.searchParams.set('p', last.page); node.href = route.href; }
      if (node._rpmcHtml !== html) { node.innerHTML = html; node._rpmcHtml = html; }
      node.classList.toggle('is-unread', !!group.unread);
      retained.add(node);
      if (area.children[i] !== node) area.insertBefore(node, area.children[i] || null);
    });
    Array.from(area.children).forEach(function (node) { if (!retained.has(node)) node.remove(); });
    if (!groups.length) {
      var empty = document.createElement('p'); empty.className = 'rpmc-dialog-empty';
      empty.textContent = state.listFilter || state.unreadOnly ? '\u041f\u043e\u0434\u0445\u043e\u0434\u044f\u0449\u0438\u0445 \u0434\u0438\u0430\u043b\u043e\u0433\u043e\u0432 \u043d\u0435\u0442.' : '\u041f\u043e\u043a\u0430 \u043d\u0435\u0442 \u043f\u0435\u0440\u0435\u043f\u0438\u0441\u043e\u043a.';
      area.appendChild(empty);
    }
    hydrateAvatars(area);
    historyStatus();
  }
  async function refreshDialogues(incremental, seed, options) {
    if (state.refresh) return state.refresh;
    state.refresh = (async function () {
      if (!(await apiRecent(false))) await allMailboxRows(Object.assign({}, options, { incremental: incremental, seed: seed }));
      renderDialogues();
      $('.rpmc-list-status', state.list).textContent = state.warnings.length ?
        '\u041d\u0435 \u0432\u0441\u0435 \u043f\u0430\u043f\u043a\u0438 \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u043b\u0438\u0441\u044c. \u041c\u043e\u0436\u043d\u043e \u043f\u043e\u0432\u0442\u043e\u0440\u0438\u0442\u044c \u0447\u0435\u0440\u0435\u0437 \u21bb \u0438\u043b\u0438 \u043e\u0442\u043a\u0440\u044b\u0442\u044c \u043e\u0431\u044b\u0447\u043d\u044b\u0435 \u041b\u0421.' : '';
    })();
    try { await state.refresh; } catch (error) { state.warnings.push(error.message); state.historyError = error.message; historyStatus(); } finally { state.refresh = null; startHistorySync(); }
  }
  function hideNative() {
    var post = nativePost(document);
    var nodes = [post];
    $$('#pun-main h1, #pun-messages h1').forEach(function (heading) {
      if (!heading.closest('#resonance-pm-chat') && cleanSubject(heading.textContent).toLowerCase() === state.subject.toLowerCase()) nodes.push(heading);
    });
    if (post && post.parentElement) {
      Array.from(post.parentElement.children).forEach(function (el) {
        if (el.matches('.linkst, .linksb, .post-links, .formsubmit')) nodes.push(el);
      });
    }
    nodes.forEach(rememberHidden);
    hideMailboxNodes();
  }
  function showNative() {
    state.hidden.forEach(function (item) {
      if (item.value) item.node.style.setProperty('display', item.value, item.priority);
      else item.node.style.removeProperty('display');
    });
    state.hidden = [];
  }
  function messageScrollState(area) {
    if (area._rpmcScroll) return area._rpmcScroll;
    var stateScroll = { area: area, follow: area.scrollHeight - area.scrollTop - area.clientHeight < 12,
      animation: null, generation: 0, internal: false, anchor: null, anchorTop: 0, observed: new Set() };
    area._rpmcScroll = stateScroll;
    area.style.scrollBehavior = 'auto';
    function remember() {
      var top = typeof area.getBoundingClientRect === 'function' ? area.getBoundingClientRect().top : 0;
      stateScroll.anchor = Array.from(area.children).find(function (row) {
        return typeof row.getBoundingClientRect === 'function' && row.getBoundingClientRect().bottom > top + 1;
      }) || null;
      stateScroll.anchorTop = stateScroll.anchor ? stateScroll.anchor.getBoundingClientRect().top - top : 0;
    }
    stateScroll.remember = remember;
    function cancel() {
      stateScroll.generation++;
      if (stateScroll.animation != null && typeof cancelAnimationFrame === 'function') cancelAnimationFrame(stateScroll.animation);
      stateScroll.animation = null;
    }
    stateScroll.cancel = cancel;
    area.addEventListener('scroll', function () {
      if (!stateScroll.internal && stateScroll.animation == null) stateScroll.follow = area.scrollHeight - area.scrollTop - area.clientHeight < 12;
      remember();
    }, { passive: true });
    ['wheel', 'touchstart', 'pointerdown', 'keydown'].forEach(function (type) {
      area.addEventListener(type, function () { cancel(); stateScroll.follow = false; remember(); }, { passive: true });
    });
    function resized() {
      if (!area.isConnected) return;
      if (stateScroll.follow) {
        var target = Math.max(0, area.scrollHeight - area.clientHeight);
        if (stateScroll.lastAppend && Date.now() - stateScroll.lastAppend < 1500 && Math.abs(target - area.scrollTop) > 1) scrollMessagesToEnd(area, true);
        else if (stateScroll.animation == null) instantMessageScroll(stateScroll, target);
      }
      else if (stateScroll.anchor && stateScroll.anchor.isConnected) {
        var top = area.getBoundingClientRect().top;
        instantMessageScroll(stateScroll, area.scrollTop + stateScroll.anchor.getBoundingClientRect().top - top - stateScroll.anchorTop);
      }
      remember();
    }
    area.addEventListener('load', resized, true); area.addEventListener('loadedmetadata', resized, true);
    area.addEventListener('toggle', resized, true);
    if (typeof ResizeObserver !== 'undefined') {
      stateScroll.observer = new ResizeObserver(resized); stateScroll.observer.observe(area);
    }
    remember(); return stateScroll;
  }
  function instantMessageScroll(scroll, top) {
    scroll.internal = true; scroll.area.scrollTop = Math.max(0, top); scroll.internal = false;
  }
  function scrollMessagesToEnd(area, smooth) {
    var scroll = messageScrollState(area); scroll.cancel(); scroll.follow = true;
    if (!smooth || typeof area.scrollTo !== 'function' || typeof cancelAnimationFrame !== 'function' ||
        (typeof window.matchMedia === 'function' && window.matchMedia('(prefers-reduced-motion: reduce)').matches)) {
      instantMessageScroll(scroll, area.scrollHeight - area.clientHeight); scroll.remember(); return;
    }
    var generation = scroll.generation, from = area.scrollTop, started = null;
    function step(time) {
      if (generation !== scroll.generation || !area.isConnected) return;
      if (started == null) started = time;
      var progress = Math.min(1, (time - started) / 240), ease = 1 - Math.pow(1 - progress, 3);
      var target = Math.max(0, area.scrollHeight - area.clientHeight);
      instantMessageScroll(scroll, from + (target - from) * ease);
      scroll.remember();
      if (progress < 1) scroll.animation = requestAnimationFrame(step);
      else { scroll.animation = null; instantMessageScroll(scroll, target); scroll.remember(); }
    }
    scroll.animation = requestAnimationFrame(step);
  }
  function mountChat(partner) {
    var post = nativePost(document); if (!post) throw new Error('\u041d\u0435 \u043d\u0430\u0439\u0434\u0435\u043d\u043e \u043e\u0442\u043a\u0440\u044b\u0442\u043e\u0435 \u043f\u0438\u0441\u044c\u043c\u043e');
    var chat = document.createElement('section'); chat.id = 'resonance-pm-chat';
    chat.setAttribute('aria-label', '\u041f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0430 \u0441 ' + partner.name);
    chat.innerHTML = '<div class="rpmc-shell"><header class="rpmc-head">' +
      '<a class="rpmc-back" href="' + esc(chatUrl(mailboxUrl('0', 1))) + '" title="\u041a \u0441\u043f\u0438\u0441\u043a\u0443 \u0434\u0438\u0430\u043b\u043e\u0433\u043e\u0432">‹</a>' +
      avatarHtml(partner, 'rpmc-head-avatar') + '<div class="rpmc-person"><span class="rpmc-name">' + esc(partner.name) + '</span>' +
      (partner.href ? '<a class="rpmc-profile" href="' + esc(partner.href) + '">\u043f\u0440\u043e\u0444\u0438\u043b\u044c</a>' : '') + '</div>' +
      '<span class="rpmc-count"></span><button class="rpmc-refresh" type="button" title="\u041e\u0431\u043d\u043e\u0432\u0438\u0442\u044c \u043f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0443">\u21bb</button>' +
      '<a class="rpmc-native" href="' + esc(modeLink(location.href, 'native')) + '">\u043e\u0431\u044b\u0447\u043d\u044b\u0435 \u041b\u0421</a></header>' +
      '<div class="rpmc-history-status" role="status"></div>' +
      '<div class="rpmc-messages" role="region" aria-label="\u0421\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f"></div>' +
      '<button class="rpmc-new-incoming" type="button" hidden>\u041d\u043e\u0432\u044b\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u044f \u2193</button>' +
      '<div class="rpmc-composer"></div>' +
      '<div class="rpmc-status" role="status" aria-live="polite"></div></div>';
    // Mount outside a native deletion form; nested forms break submission.
    var anchor = post.closest('form') || post;
    anchor.parentNode.insertBefore(chat, anchor);
    state.chat = chat; addHistoryControls(chat);
    var ds = document.createElement('div'); ds.className = 'rpmc-draft-state'; ds.setAttribute('role', 'status'); $('.rpmc-composer', chat).after(ds);
    $('.rpmc-refresh', chat).addEventListener('click', function () { refreshHistory(true); });
    $('.rpmc-new-incoming', chat).addEventListener('click', function () {
      scrollMessagesToEnd($('.rpmc-messages', chat), true); this.hidden = true;
    });
    $('.rpmc-messages', chat).addEventListener('scroll', function () {
      markVisibleApiRead();
      if (this.scrollHeight - this.scrollTop - this.clientHeight < 90) $('.rpmc-new-incoming', chat).hidden = true;
    }, { passive: true });
    bindQuoting(chat);
    return chat;
  }
  function renderMessages(messages, scrollToEnd) {
    var area = $('.rpmc-messages', state.chat);
    var scrolling = messageScrollState(area), nearBottom = scrolling.follow;
    var oldScroll = area.scrollTop, previous = new Map(), retained = new Set(), hasIncoming = false;
    var firstPaint = !area.children.length && oldScroll === 0, hasNewer = false;
    var lastId = Math.max(0, ...Array.from(area.children).map(function (row) { return Number(row.dataset.messageId) || 0; }));
    scrolling.remember();
    var anchor = null, anchorTop = 0;
    if (!nearBottom && !scrollToEnd && typeof area.getBoundingClientRect === 'function') {
      var top = area.getBoundingClientRect().top;
      anchor = Array.from(area.children).find(function (row) {
        return typeof row.getBoundingClientRect === 'function' && row.getBoundingClientRect().bottom > top + 1;
      });
      if (anchor) anchorTop = anchor.getBoundingClientRect().top;
    }
    Array.from(area.children).forEach(function (row) { previous.set(row.dataset.messageKey, row); });
    var totalMessages = messages.length;
    var end = state.displayBefore == null ? messages.length : Math.min(state.displayBefore, messages.length);
    messages = messages.slice(Math.max(0, end - Math.min(DISPLAY_LIMIT, state.displayCount)), end);
    messages.forEach(function (msg, i) {
      var key = msg.direction + ':' + msg.id, row = previous.get(key);
      if (!row) {
        row = document.createElement('div'); row.className = 'rpmc-row ' + msg.direction;
        row.dataset.messageId = String(msg.id); row.dataset.messageKey = key;
        if (msg.id > lastId) { hasNewer = true; if (msg.direction === 'in') hasIncoming = true; }
      }
      var html = '<div class="rpmc-bubble-wrap"><div class="rpmc-bubble">' + msg.html + '</div><div class="rpmc-meta">' +
        esc(msg.date || msg.dateText || '') + ' \xb7 <a class="rpmc-open-original" href="' + esc(nativeUrl(msg.href)) + '">#' + msg.id + '</a>' +
        '<button type="button" class="rpmc-quote-button" data-quote-message="' + msg.id + '">\u0426\u0438\u0442\u0438\u0440\u043e\u0432\u0430\u0442\u044c</button></div></div>';
      // Unchanged bubbles remain the same DOM nodes: polling keeps selection,
      // open quotes and playing audio/video intact.
      if (row._rpmcHtml !== html) {
        row.innerHTML = (msg.direction === 'in' ? avatarHtml(state.partner, 'rpmc-mini-avatar') : '') + html; row._rpmcHtml = html;
      }
      retained.add(row);
      if (area.children[i] !== row) area.insertBefore(row, area.children[i] || null);
    });
    reconcileOutgoing(messages);
    renderOutgoing(area, retained);
    Array.from(area.children).forEach(function (row) {
      if (!retained.has(row)) {
        if (scrolling.observer) scrolling.observer.unobserve(row);
        scrolling.observed.delete(row); row.remove();
      } else if (scrolling.observer && !scrolling.observed.has(row)) {
        scrolling.observed.add(row); scrolling.observer.observe(row);
      }
    });
    hydrateAvatars(state.chat);
    historyStatus();
    setTimeout(markVisibleApiRead, 300);
    $('.rpmc-count', state.chat).textContent = totalMessages + ' ' + wordMessages(totalMessages);
    var incoming = $('.rpmc-new-incoming', state.chat);
    if (incoming && hasIncoming && !scrollToEnd && !nearBottom) incoming.hidden = false;
    if (hasNewer && !firstPaint) scrolling.lastAppend = Date.now();
    if (scrollToEnd || nearBottom) {
      if (firstPaint || (!hasNewer && !scrollToEnd)) {
        if (scrolling.animation == null) scrollMessagesToEnd(area, false);
      } else scrollMessagesToEnd(area, true);
    } else {
      instantMessageScroll(scrolling, anchor && anchor.isConnected ? oldScroll + anchor.getBoundingClientRect().top - anchorTop : oldScroll);
      scrolling.remember();
    }
  }
  async function refreshHistory(scrollToEnd, options) {
    options = options || {};
    if (state.refresh && !options.afterSend) return state.refresh;
    while (state.refresh) await state.refresh;
    state.refresh = (async function () {
      var info = $('.rpmc-history-status', state.chat);
      if (!options.quiet) info.textContent = '\u041e\u0431\u043d\u043e\u0432\u043b\u044f\u044e \u043f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0443\u2026';
      if (!options.rows && await apiHistory(false, scrollToEnd)) return state.rows;
      var all = options.rows || await allMailboxRows(options);
      var current = state.current || state.rows.find(function (e) { return e.id === currentMessageId; });
      var entries = historyEntries(all, current).slice(-Math.max(30, state.displayCount));
      var cached = entries.filter(function (e) { return memory.has(e.direction + ':' + e.id); }).map(function (e) {
        return Object.assign({}, memory.get(e.direction + ':' + e.id), e, { html: memory.get(e.direction + ':' + e.id).html, date: e.dateText || memory.get(e.direction + ':' + e.id).date, timestamp: e.timestamp || memory.get(e.direction + ':' + e.id).timestamp });
      });
      mergeVisible(cached, scrollToEnd); if (cached.length) scrollToEnd = false;
      // Fresh replies have foreground priority. Older missing letters are handled
      // by a separate, resumable pass, so they cannot block sends or new replies.
      var missing = entries.filter(function (e) { return !memory.has(e.direction + ':' + e.id); }).reverse();
      missing = missing.slice(0, options.incremental ? 6 : 20);
      await mapLimit(missing, 1, async function (entry) {
        try { mergeVisible([await loadMessage(entry)], scrollToEnd); scrollToEnd = false; }
        catch (_) {
          state.warnings.push('\u041d\u0435 \u0437\u0430\u0433\u0440\u0443\u0437\u0438\u043b\u043e\u0441\u044c \u043f\u0438\u0441\u044c\u043c\u043e #' + entry.id + '.');
        }
      });
      if (!options.incremental) state.hadPartialHistory = !!state.warnings.length;
      state.historyError = state.warnings.length ? '\u041d\u0435 \u0432\u0441\u0451 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043e\u0431\u043d\u043e\u0432\u0438\u0442\u044c. \u041d\u0430\u0436\u043c\u0438 \u21bb, \u0447\u0442\u043e\u0431\u044b \u043f\u043e\u0432\u0442\u043e\u0440\u0438\u0442\u044c.' : '';
      historyStatus();
      return state.rows;
    })();
    try { return await state.refresh; }
    catch (error) { state.warnings.push(error.message); state.historyError = error.message; historyStatus(); return state.rows; }
    finally { state.refresh = null; startHistorySync(); }
  }
  function canPoll() {
    return !state.stopped && state.mode === 'chat' && !document.hidden &&
      !(typeof navigator !== 'undefined' && navigator.onLine === false) && !!(state.chat || state.list);
  }
  function pollInterval() {
    if (state.pollErrors) return Math.min(300000, POLL_INTERVAL * Math.pow(2, Math.min(state.pollErrors, 4)));
    return state.incomingHint ? NOTICE_POLL_GAP : POLL_INTERVAL;
  }
  function requestIncomingRefresh() {
    if (Date.now() - state.lastHint < NOTICE_POLL_GAP) return;
    state.lastHint = Date.now(); state.incomingHint = true;
    if (canPoll()) schedulePoll(0);
  }
  function publishMailboxUpdate() {
    if (state.updatesChannel) try { state.updatesChannel.postMessage({ type: 'invalidate', uid: Number(window.UserID), at: Date.now() }); } catch (_) {}
  }

  function setupMailboxUpdates() {
    if (state.updatesChannel || !accountCacheKey() || typeof window.BroadcastChannel !== 'function') return;
    try {
      state.updatesChannel = new window.BroadcastChannel('resonance-pm-updates-v14:' + Number(window.UserID));
      state.updatesChannel.onmessage = function (event) {
        var value = event.data;
        if (!value || value.type !== 'invalidate' || value.uid !== Number(window.UserID) || !Number.isFinite(value.at)) return;
        state.incomingHint = true; schedulePoll();
      };
    } catch (_) {}
  }

  function schedulePoll(delay) {
    clearTimeout(state.pollTimer);
    if (!canPoll()) return;
    var interval = pollInterval();
    var next = Math.max(delay == null ? interval : delay,
      state.lastPoll + interval - Date.now(), networkPause() - Date.now());
    state.pollTimer = setTimeout(pollOnce, Math.max(1000, next));
  }
  async function pollOnce() {
    if (!canPoll()) return;
    if (state.pollBusy || state.refresh || state.submitting || state.historyBusy) { schedulePoll(1000); return; }
    var recent = Math.max(state.lastPoll, Number(readLocalState('rpmc-last-poll:' + String(window.UserID) + ':' + (state.partner ? state.partner.id : 'list') + ':' + draftTab)) || 0);
    var interval = pollInterval();
    if (Date.now() - recent < interval) { schedulePoll(interval - (Date.now() - recent)); return; }
    state.pollBusy = true;
    try {
      async function refreshIfLeader(lock) {
        if (lock === null || !canPoll()) return;
        var last = Number(readLocalState('rpmc-last-poll:' + String(window.UserID) + ':' + (state.partner ? state.partner.id : 'list') + ':' + draftTab)) || 0;
        if (Date.now() - last < pollInterval()) return;
        state.lastPoll = Date.now(); writeLocalState('rpmc-last-poll:' + String(window.UserID) + ':' + (state.partner ? state.partner.id : 'list') + ':' + draftTab, state.lastPoll);
        state.incomingHint = false;
        var boxes = ['0'];
        // Outgoing folders only need an occasional cross-tab resync; sending
        // from this conversation already refreshes them on confirmation.
        state.pollRound++;
        var sentBoxes = mailboxBoxes().filter(function (box) { return direction(box) === 'out'; });
        if (state.pollRound % 4 === 0 && sentBoxes.length) boxes.push(sentBoxes[Math.floor(state.pollRound / 4 - 1) % sentBoxes.length]);
        var options = { incremental: true, quiet: true, fresh: true, boxes: boxes };
        if (state.chat) await refreshHistory(false, options);
        else if (state.list) await refreshDialogues(true, null, options);
        state.pollErrors = state.warnings.length ? state.pollErrors + 1 : 0;
        if (!state.pollErrors) publishMailboxUpdate();
      }
      if (typeof navigator !== 'undefined' && navigator.locks && navigator.locks.request) {
        await navigator.locks.request('resonance-pm-poll-v14:' + String(window.UserID) + ':' + (state.partner ? state.partner.id : 'list'), { ifAvailable: true }, refreshIfLeader);
      } else { await refreshIfLeader(true); }
    } catch (_) { state.pollErrors++; }
    finally {
      state.pollBusy = false;
      schedulePoll(Math.min(300000, POLL_INTERVAL * Math.pow(2, Math.min(state.pollErrors, 4))));
    }
  }
  function startPolling() {
    if (state.pollStarted) return;
    state.pollStarted = true; state.lastPoll = Date.now();
    setupMailboxUpdates();
    window.ResonancePmIncomingHint = requestIncomingRefresh;
    function noticeHint(node, fresh) { if (fresh) requestIncomingRefresh(); }
    state.noticeObserver = watchPmNotices(window, document, noticeHint);
    document.addEventListener('visibilitychange', function () { saveDraftNow(); schedulePoll(document.hidden ? POLL_INTERVAL : 0); if (!document.hidden) startHistorySync(); });
    window.addEventListener('focus', function () { if (!state.pollBusy) schedulePoll(0); startHistorySync(); });
    window.addEventListener('online', function () { schedulePoll(0); startHistorySync(); });
    window.addEventListener('pagehide', function () {
      saveDraftNow(); state.stopped = true; clearTimeout(state.pollTimer); saveCacheNow();
      if (state.updatesChannel) { state.updatesChannel.close(); state.updatesChannel = null; }
    });
    window.addEventListener('pageshow', function (event) {
      state.stopped = false; setupMailboxUpdates(); schedulePoll(); startHistorySync();
      if (event.persisted) state.noticeObserver = watchPmNotices(window, document, noticeHint);
    });
    schedulePoll();
  }
  function pendingText(value) {
    return normalizeText(String(value || '')
      .replace(/\[(?:img|audio|video)(?:=[^\]]*)?\][\s\S]*?\[\/(?:img|audio|video)\]/gi, '')
      .replace(/\[\/?(?:b|i|u|s|strike|color|font|size|align|url|quote|spoiler|hide|code|list|table|tr|td|sup|sub)(?:=[^\]]*)?\]/gi, ''));
  }
  function deliveredText(message) { return messageParts(message).text; }
  function reconcileOutgoing(messages) {
    var removed = new Set();
    state.outgoing.forEach(function (item) {
      // History is NOT an acknowledgement for an unknown POST. Identical texts
      // in two tabs cannot be disambiguated without a server operation key.
      if (item.status !== 'sent') return;
      var candidates = messages.filter(function (message) {
        if (message.direction !== 'out' || message.loadFailed || !matchesPartner(message, state.partner)) return false;
        if (item.serverId) return message.id === item.serverId;
        return message.id > item.maxId && !item.before.has(message.id) && !state.confirmedOutgoingIds.has(message.id) &&
          deliveredText(message) === pendingText(item.text) &&
          JSON.stringify(mediaInMessage(message)) === JSON.stringify(mediaInDraft(item.text));
      });
      if (candidates.length !== 1) return;
      state.confirmedOutgoingIds.add(candidates[0].id); removed.add(item);
    });
    if (removed.size) { state.outgoing = state.outgoing.filter(function (item) { return !removed.has(item); }); scheduleDraftSave(); }
  }

  function renderOutgoing(area, retained) {
    state.outgoing.forEach(function (item) {
      var row = item.node;
      if (!row) {
        row = document.createElement('div'); row.className = 'rpmc-row out rpmc-pending-row';
        row.setAttribute('data-pending-message', String(item.id));
        var wrap = document.createElement('div'); wrap.className = 'rpmc-bubble-wrap';
        var bubble = document.createElement('div'); bubble.className = 'rpmc-bubble rpmc-pending-text'; bubble.textContent = item.text;
        var meta = document.createElement('div'); meta.className = 'rpmc-delivery-state'; meta.setAttribute('role', 'status');
        var restore = document.createElement('button'); restore.type = 'button'; restore.className = 'rpmc-operation-action'; restore.textContent = '\u041e\u0442\u043a\u0440\u044b\u0442\u044c \u0442\u0435\u043a\u0441\u0442';
        restore.addEventListener('click', function () {
          var existing = $('.rpmc-operation-copy', wrap);
          if (existing) { existing.remove(); return; }
          var copy = document.createElement('textarea'); copy.className = 'rpmc-operation-copy'; copy.value = item.text; copy.readOnly = true;
          copy.setAttribute('aria-label', '\u0422\u0435\u043a\u0441\u0442 \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0438 \u0434\u043b\u044f \u043a\u043e\u043f\u0438\u0440\u043e\u0432\u0430\u043d\u0438\u044f'); wrap.appendChild(copy); copy.focus(); copy.select();
        });
        var check = document.createElement('button'); check.type = 'button'; check.className = 'rpmc-operation-action'; check.textContent = '\u041f\u0440\u043e\u0432\u0435\u0440\u0438\u0442\u044c \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0443';
        check.addEventListener('click', async function () {
          check.disabled = true;
          try {
            await refreshHistory(false, { fresh: true, afterSend: true, boxes: mailboxBoxes().filter(function (box) { return direction(box) === 'out'; }) });
            if (item.status === 'unknown') setStatus('\u0418\u0441\u0442\u043e\u0440\u0438\u044f \u043e\u0431\u043d\u043e\u0432\u043b\u0435\u043d\u0430. \u0421\u0440\u0430\u0432\u043d\u0438 \u043f\u0438\u0441\u044c\u043c\u043e \u0441 \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u043d\u044b\u043c \u0442\u0435\u043a\u0441\u0442\u043e\u043c. \u0411\u0435\u0437 \u043e\u0442\u0432\u0435\u0442\u0430 \u0441\u0435\u0440\u0432\u0435\u0440\u0430 \u043f\u043e\u0434\u0442\u0432\u0435\u0440\u0434\u0438\u0442\u044c \u0438\u043c\u0435\u043d\u043d\u043e \u044d\u0442\u0443 \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0443 \u043d\u0435\u043b\u044c\u0437\u044f.', true);
          } catch (_) { setStatus('\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043e\u0431\u043d\u043e\u0432\u0438\u0442\u044c \u0438\u0441\u0442\u043e\u0440\u0438\u044e. \u0422\u0435\u043a\u0441\u0442 \u0434\u043e\u0441\u0442\u0443\u043f\u0435\u043d \u043f\u043e \u043a\u043d\u043e\u043f\u043a\u0435 \u00ab\u041e\u0442\u043a\u0440\u044b\u0442\u044c \u0442\u0435\u043a\u0441\u0442\u00bb.', true); }
          finally { check.disabled = false; }
        });
        wrap.append(bubble, meta, restore, check); row.appendChild(wrap); item.node = row;
      }
      row.setAttribute('data-delivery', item.status);
      $('.rpmc-delivery-state', row).textContent = item.status === 'sending' ? '\u041e\u0442\u043f\u0440\u0430\u0432\u043b\u044f\u0435\u0442\u0441\u044f\u2026' : item.status === 'sent' ? '\u041e\u0442\u043f\u0440\u0430\u0432\u043b\u0435\u043d\u043e' :
        item.status === 'failed' ? '\u041d\u0435 \u043e\u0442\u043f\u0440\u0430\u0432\u043b\u0435\u043d\u043e' : '\u0420\u0435\u0437\u0443\u043b\u044c\u0442\u0430\u0442 \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0438 \u043d\u0435\u0438\u0437\u0432\u0435\u0441\u0442\u0435\u043d';
      row.title = item.error || ''; retained.add(row);
      if (row.parentNode !== area || area.lastElementChild !== row) area.appendChild(row);
    });
  }

  function startOutgoing(pending) {
    if (pending.optimistic) return;
    var item = { id: pending.id, started: pending.started, text: pending.draft, status: 'sending',
      maxId: pending.maxId, before: new Set(pending.before), node: null };
    pending.optimistic = item; state.outgoing.push(item); saveDraftNow();
    renderMessages(state.rows, true);
  }
  function installSendFeedback(form) {
    $$('input[type="submit"], button[type="submit"], button:not([type])', form).forEach(function (button) {
      if (isPreview(button) || button.closest('.rpmc-send-control')) return;
      var holder = form.ownerDocument.createElement('span'); holder.className = 'rpmc-send-control';
      button.parentNode.insertBefore(holder, button); holder.appendChild(button);
      var feedback = form.ownerDocument.createElement('span'); feedback.className = 'rpmc-send-feedback';
      feedback.textContent = '\u041e\u0442\u043f\u0440\u0430\u0432\u043b\u044f\u0435\u0442\u0441\u044f\u2026'; feedback.setAttribute('aria-hidden', 'true');
      holder.appendChild(feedback);
    });
  }
  function sendFeedback(pending, busy) {
    if (!pending.form) return;
    $$('.rpmc-send-control', pending.form).forEach(function (holder) { holder.classList.toggle('is-sending', busy); });
  }
  function watchSendResponse(pending) {
    // Read the server response as soon as its DOM is ready, without waiting for
    // all forum banners, images and third-party scripts to finish loading.
    var deadline = Date.now() + 60000;
    async function check() {
      if (pending.stage !== 'submitting' || !pending.receiver || !pending.receiver.isConnected) return;
      try {
        var doc = pending.receiver.contentDocument;
        if (doc && doc.body && (doc.readyState === 'interactive' || doc.readyState === 'complete' || explicitSendSuccess(doc))) {
          await handleSendLoad(pending.receiver);
        }
      } catch (_) {}
      // This reads the local iframe DOM, not the network. Bound it anyway;
      // a late native load event can still confirm the original POST.
      if (state.pending === pending && Date.now() < deadline) pending.responseTimer = setTimeout(check, 150);
    }
    pending.responseTimer = setTimeout(check, 150);
  }

  function quoteAuthor(message) {
    if (message.direction === 'in') return state.partner.name;
    return typeof window.UserLogin === 'string' ? window.UserLogin : '';
  }
  function quoteCode(text, author) {
    author = String(author || '').replace(/[\[\]"\r\n]/g, '').trim();
    return '[quote' + (author ? '="' + author + '"' : '') + ']\n' + String(text).trim() + '\n[/quote]\n';
  }
  function selectedMessageQuote(selection) {
    if (!selection || selection.isCollapsed || !selection.rangeCount) return null;
    var range = selection.getRangeAt(0);
    var start = range.startContainer.nodeType === 1 ? range.startContainer : range.startContainer.parentElement;
    var bubble = start && start.closest('.rpmc-bubble');
    if (!bubble || !state.chat.contains(bubble) || !bubble.contains(range.endContainer)) return null;
    var row = bubble.closest('.rpmc-row');
    var message = state.rows.find(function (e) { return e.id === Number(row.dataset.messageId); });
    var text = selection.toString().trim();
    if (!message || !text) return null;
    return { id: message.id, text: text, author: quoteAuthor(message), rect: range.getBoundingClientRect() };
  }
  function insertQuote(text, author) {
    var simple = state.chat && $('.rpmc-simple-textarea', state.chat);
    if (simple) {
      if (state.submitting) { setStatus('Дождись завершения текущей отправки.', true); return; }
      try {
        simpleEditorAdapter(simple).insertQuote(quoteCode(text, author));
        scheduleDraftSave();
        try { simple.focus({ preventScroll: true }); } catch (_) { simple.focus(); }
        simple.scrollIntoView({ block: 'nearest' });
        setStatus('');
      } catch (error) { setStatus(error.message, true); }
      return;
    }
    var inline = state.chat && postForm(state.chat);
    if (inline && inline.form.classList.contains('rpmc-native-inline')) {
      if (state.submitting) { setStatus('Дождись завершения текущей отправки.', true); return; }
      try {
        editorAdapter(window, inline.textarea).insertQuote(quoteCode(text, author));
        scheduleDraftSave(); try { inline.textarea.focus({ preventScroll: true }); } catch (_) { inline.textarea.focus(); }
        inline.form.scrollIntoView({ block: 'nearest' }); setStatus('');
      } catch (error) { setStatus(error.message, true); }
      return;
    }
    var found = null, frame = state.frame;
    try { found = frame && postForm(frame.contentDocument); } catch (_) {}
    if (!found || !state.editorReady || state.submitting || frame.style.visibility === 'hidden') {
      state.queuedQuotes.push({ text: text, author: author }); setStatus('Цитата будет добавлена после загрузки редактора.'); return;
    }
    try {
      editorAdapter(frame, found.textarea).insertQuote(quoteCode(text, author));
      scheduleDraftSave(); found.textarea.focus({ preventScroll: true });
      frame.scrollIntoView({ block: 'nearest' }); setStatus('');
    } catch (error) { setStatus(error.message, true); }
  }

  function bindQuoting(chat) {
    var picker = document.createElement('button');
    picker.type = 'button'; picker.className = 'rpmc-selection-quote'; picker.hidden = true;
    picker.textContent = '\u0426\u0438\u0442\u0438\u0440\u043e\u0432\u0430\u0442\u044c'; picker.setAttribute('aria-label', '\u0426\u0438\u0442\u0438\u0440\u043e\u0432\u0430\u0442\u044c \u0432\u044b\u0434\u0435\u043b\u0435\u043d\u043d\u044b\u0439 \u0444\u0440\u0430\u0433\u043c\u0435\u043d\u0442');
    chat.appendChild(picker);
    function rememberSelection() {
      var selected = selectedMessageQuote(window.getSelection());
      state.selectedQuote = selected; picker.hidden = !selected;
      if (!selected) return;
      picker.style.left = Math.max(8, Math.min(selected.rect.left, window.innerWidth - 140)) + 'px';
      picker.style.top = Math.max(8, Math.min(selected.rect.bottom + 7, window.innerHeight - 42)) + 'px';
    }
    $('.rpmc-messages', chat).addEventListener('mouseup', rememberSelection);
    $('.rpmc-messages', chat).addEventListener('keyup', rememberSelection);
    document.addEventListener('selectionchange', function () {
      var selection = window.getSelection();
      if (!selection || selection.isCollapsed) picker.hidden = true;
    });
    picker.addEventListener('mousedown', function (e) { e.preventDefault(); });
    picker.addEventListener('click', function () {
      var selected = state.selectedQuote; if (!selected) return;
      picker.hidden = true; state.selectedQuote = null;
      insertQuote(selected.text, selected.author);
      var selection = window.getSelection(); if (selection) selection.removeAllRanges();
    });
    chat.addEventListener('click', function (e) {
      var button = e.target.closest('[data-quote-message]'); if (!button) return;
      var id = Number(button.dataset.quoteMessage);
      var message = state.rows.find(function (row) { return row.id === id; }); if (!message) return;
      var selected = selectedMessageQuote(window.getSelection());
      if (selected && selected.id === id) { insertQuote(selected.text, selected.author); }
      else {
        var bubble = $('.rpmc-bubble', button.closest('.rpmc-row'));
        var copy = bubble.cloneNode(true);
        $$('.rpmc-quote', copy).forEach(function (quote) { quote.remove(); });
        $$('br', copy).forEach(function (br) { br.replaceWith('\n'); });
        $$('p, div, li', copy).forEach(function (block) { block.appendChild(document.createTextNode('\n')); });
        var text = copy.textContent.trim() || bubble.textContent.trim();
        if (text) insertQuote(text, quoteAuthor(message));
      }
      picker.hidden = true; state.selectedQuote = null;
    });
    chat.addEventListener('mousedown', function (e) {
      if (e.target.closest('[data-quote-message]')) e.preventDefault();
    });
    $('.rpmc-messages', chat).addEventListener('scroll', function () { picker.hidden = true; }, { passive: true });
    window.addEventListener('scroll', function () { picker.hidden = true; }, { passive: true });
  }

  async function composeHref() {
    // CF Messenger 0.9.7 uses this native RusFF route directly. Prefer it over
    // fetching a profile page just to discover the same link: the editor must
    // remain available even when profile/history requests fail.
    var partnerId = Number(state.partner && state.partner.id);
    if (Number.isSafeInteger(partnerId) && partnerId > 0) return '/messages.php?action=new&uid=' + encodeURIComponent(partnerId);
    var incoming = state.rows.filter(function (e) { return e.direction === 'in' && e.reply; }).pop();
    if (incoming) return incoming.reply;
    if (direction(currentBox) === 'in') { var direct = replyHref(document); if (direct) return direct; }
    if (state.partner && state.partner.href) {
      var profile = await fetchDoc(state.partner.href);
      var link = $$('a[href*="messages.php"]', profile).find(function (a) {
        var u = localUrl(a.getAttribute('href'));
        return u && !/delete|remove|clear/i.test(u.searchParams.get('action') || '') &&
          /\u043d\u0430\u043f\u0438\u0441\u0430\u0442\u044c.*(\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435|\u043f\u0438\u0441\u044c\u043c\u043e)|\u043e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c.*(\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435|\u043f\u0438\u0441\u044c\u043c\u043e)/i.test(a.textContent);
      });
      if (link) return localUrl(link.getAttribute('href')).href;
    }
    throw new Error('\u041d\u0435 \u043d\u0430\u0448\u043b\u0430\u0441\u044c \u0441\u0441\u044b\u043b\u043a\u0430 \u043e\u0442\u0432\u0435\u0442\u0430 \u044d\u0442\u043e\u043c\u0443 \u0441\u043e\u0431\u0435\u0441\u0435\u0434\u043d\u0438\u043a\u0443');
  }

  function isolateEditor(doc, keep) {
    var style = doc.createElement('style'); style.id = 'rpmc-native-editor-style';
    style.textContent = `
      html, body { background: transparent !important; min-width: 0 !important; width: 100% !important; height: auto !important; min-height: 0 !important; margin: 0 !important; padding: 0 !important; overflow-x: hidden !important; }
      .rpmc-editor-hidden { display: none !important; }
      .rpmc-editor-path { display: block !important; width: auto !important; min-width: 0 !important; max-width: none !important; height: auto !important; min-height: 0 !important; margin: 0 !important; padding: 0 !important; border: 0 !important; box-shadow: none !important; background: transparent !important; float: none !important; position: static !important; }
      .rpmc-editor-path::before, .rpmc-editor-path::after { display: none !important; }
      #post, #post-form { margin: 0 !important; padding: 8px !important; width: auto !important; border: 0 !important; border-radius: 12px !important; background: transparent !important; }
      #main-reply, textarea[name="req_message"] { box-sizing: border-box !important; min-height: 150px !important; max-width: 100% !important; width: 100% !important; border-radius: 0 0 12px 12px !important; }
      #form-buttons { width: 100% !important; max-width: 100% !important; margin: 0 !important; box-sizing: border-box !important; }
      #form-buttons tr { margin-left: 0 !important; text-align: left !important; white-space: normal !important; }
      #form-buttons #button-mask { display: none !important; }
      .rpmc-native-form fieldset { min-width: 0 !important; }
      .rpmc-native-form .formsubmit { display: flex; flex-wrap: wrap; align-items: center; gap: 6px; }
      .rpmc-send-control { display: inline-block; position: relative; vertical-align: middle; }
      .rpmc-send-feedback { display: none; }
      .rpmc-send-control.is-sending > input, .rpmc-send-control.is-sending > button {
        min-width: 142px !important; color: transparent !important; text-shadow: none !important; cursor: wait !important;
      }
      .rpmc-send-control.is-sending .rpmc-send-feedback {
        display: flex; position: absolute; inset: 0; align-items: center; justify-content: center; gap: 7px;
        pointer-events: none; color: #fff; font: 600 11px/1.3 sans-serif; text-transform: none;
      }
      .rpmc-send-feedback::before {
        content: ''; flex: 0 0 11px; height: 11px; box-sizing: border-box; border: 2px solid #ffffff66;
        border-top-color: #fff; border-radius: 50%; animation: rpmc-native-spin .8s linear infinite;
      }
      @keyframes rpmc-native-spin { to { transform: rotate(360deg); } }
      @media (prefers-reduced-motion: reduce) { .rpmc-send-feedback::before { animation: none; } }
    `;
    doc.head.appendChild(style);
    var node = keep;
    while (node && node !== doc.body) {
      var parent = node.parentElement; if (!parent) break;
      Array.from(parent.children).forEach(function (sibling) {
        if (sibling !== node && !sibling.matches(EDITOR_POPUPS) && !/^(SCRIPT|STYLE|LINK)$/.test(sibling.tagName)) sibling.classList.add('rpmc-editor-hidden');
      });
      parent.classList.add('rpmc-editor-path');
      // Forum themes often set fixed #pun widths with !important.
      // Inline overrides keep the native editor inside the chat at any width.
      var pathStyles = { display: 'block', width: 'auto', 'min-width': '0', 'max-width': 'none', height: 'auto',
        'min-height': '0', margin: '0', padding: '0', border: '0', 'box-shadow': 'none', background: 'transparent',
        float: 'none', position: 'static', overflow: 'visible' };
      Object.keys(pathStyles).forEach(function (key) { parent.style.setProperty(key, pathStyles[key], 'important'); });
      node = parent;
    }
  }
  function postForm(doc) {
    if (!doc) return null;
    var fields = $$('textarea#main-reply, textarea[name="req_message"], #post textarea, #post-form textarea', doc);
    for (var i = 0; i < fields.length; i++) {
      var form = fields[i].closest('form');
      if (!form) continue;
      var action = localUrl(form.getAttribute('action') || doc.location && doc.location.href || location.href, doc.location && doc.location.href || location.href);
      if (!action || !/\/messages\.php$/i.test(action.pathname)) continue;
      // RusFF forms are POST in normal operation; do not reject a theme/plugin
      // that omits the method attribute before another script normalizes it.
      return { form: form, textarea: fields[i] };
    }
    return null;
  }
  function frameError(doc) {
    var node = $('#post-errors, .errorlist, .error-list, .formerror, #pun-error .container', doc);
    return node ? normalizeText(node.textContent).slice(0, 350) : '';
  }
  function isPreview(submitter) {
    return !!submitter && /preview|\u043f\u043e\u0441\u043c\u043e\u0442\u0440|\u043f\u0440\u0435\u0434\u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440/i.test([submitter.name, submitter.value, submitter.textContent].join(' '));
  }
  function prepareSubject(form, textarea) {
    $$('input[name="req_subject"], input[name="subject"], input[name="req_title"]', form).forEach(function (field) {
      var subject = cleanSubject(state.subject || field.value) || '\u041f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0430';
      var limit = Number(field.getAttribute('maxlength'));
      field.value = limit > 0 ? subject.slice(0, limit) : subject;
      // RusFF still requires a subject in the POST, but the chat does not need
      // a visible subject control. Keep it enabled and in its original form.
      field.type = 'hidden';
      hideInlineField(field, textarea);
      if (field.id) $$('label[for]', form).forEach(function (label) {
        if (label.htmlFor === field.id && !label.contains(textarea) && !label.querySelector('#form-buttons, .form-buttons, [contenteditable]') &&
            !$$('input:not([type="hidden"]), button, select, textarea', label).some(function (control) { return control !== field; })) {
          label.style.setProperty('display', 'none', 'important');
        }
      });
    });
  }

  function frameBusy(frame, busy) {
    frame.style.visibility = busy ? 'hidden' : 'visible';
    frame.style.minHeight = busy ? '0px' : '340px';
    if (busy) frame.style.height = '0px';
  }
  function fitFrame(frame, doc) {
    if (frame._rpmcLayout && frame._rpmcLayout.doc === doc) { frame._rpmcLayout.resize(); return; }
    if (frame._rpmcLayout) frame._rpmcLayout.dispose();
    var win = frame.contentWindow, reserve = 0, queued = false;
    function setStyle(node, property, value) {
      if (node.style.getPropertyValue(property) !== value) node.style.setProperty(property, value, 'important');
    }
    function visible(panel) {
      if (panel.hidden || panel.classList.contains('rpmc-editor-hidden')) return false;
      var style = typeof win.getComputedStyle === 'function' ? win.getComputedStyle(panel) : panel.style;
      return style.display !== 'none' && style.visibility !== 'hidden' && panel.getBoundingClientRect().height > 0;
    }
    function resize() {
      if (!frame.isConnected || frame.style.visibility === 'hidden') return;
      var form = $('.rpmc-native-form', doc);
      if (!form) return;
      var content = form.closest('#post') || form;
      var base = Math.max(340, content.getBoundingClientRect().bottom + (win.scrollY || 0) - reserve + 8);
      var panels = $$(EDITOR_POPUPS, doc).filter(visible);
      var space = panels.reduce(function (height, panel) { return Math.max(height, Math.ceil(panel.getBoundingClientRect().height) + 24); }, 0);
      var changed = space !== reserve; reserve = space;
      // Extend this same iframe upward while adding equal space inside it.
      // The form stays in place, popup events remain in their original document,
      // and the panel can cover the history without being clipped at the iframe.
      setStyle(doc.body, 'padding-top', reserve + 'px');
      setStyle(frame, 'margin-top', -reserve + 'px');
      setStyle(frame, 'position', 'relative'); setStyle(frame, 'z-index', reserve ? '50' : '1');
      setStyle(frame, 'height', Math.min(1800, base) + reserve + 'px');
      var toolbar = $('#form-buttons', doc), smile = $('#smilies-area', doc);
      if (toolbar && smile && panels.indexOf(smile) >= 0) {
        var tr = toolbar.getBoundingClientRect(), parent = smile.offsetParent || doc.documentElement, pr = parent.getBoundingClientRect();
        setStyle(smile, 'position', 'absolute'); setStyle(smile, 'transform', 'none');
        setStyle(smile, 'width', Math.max(100, Math.round(tr.width - 8)) + 'px');
        setStyle(smile, 'left', Math.round(tr.left - pr.left + 4 + (parent === doc.documentElement ? 0 : parent.scrollLeft || 0)) + 'px');
        setStyle(smile, 'top', Math.round(tr.bottom - pr.top - smile.getBoundingClientRect().height + (parent === doc.documentElement ? 0 : parent.scrollTop || 0)) + 'px');
        setStyle(smile, 'z-index', '99999');
      }
      if (changed && typeof win.dispatchEvent === 'function') win.dispatchEvent(new win.Event('resize'));
    }
    function schedule() {
      if (queued) return; queued = true;
      requestAnimationFrame(function () { queued = false; resize(); });
    }
    var sizeObserver = null, mutationObserver = null;
    if (typeof ResizeObserver !== 'undefined') {
      sizeObserver = new ResizeObserver(schedule); sizeObserver.observe(doc.body);
    }
    if (typeof MutationObserver !== 'undefined') {
      mutationObserver = new MutationObserver(function (records) {
        if (records.some(function (r) {
          return r.target.nodeType === 1 && (r.target.matches(EDITOR_POPUPS) || r.target.closest(EDITOR_POPUPS) ||
            Array.from(r.addedNodes || []).some(function (n) { return n.nodeType === 1 && (n.matches(EDITOR_POPUPS) || n.querySelector(EDITOR_POPUPS)); }));
        })) schedule();
      });
      mutationObserver.observe(doc.body, { childList: true, subtree: true, attributes: true, attributeFilter: ['style', 'class', 'hidden'] });
    }
    function outside(event) {
      if (!reserve || event.target === frame) return;
      var smile = $('#smilies-area', doc); if (smile) smile.style.display = 'none';
      schedule();
    }
    doc.addEventListener('click', schedule, true); doc.addEventListener('load', schedule, true);
    document.addEventListener('pointerdown', outside);
    frame._rpmcLayout = { doc: doc, resize: resize, dispose: function () {
      if (sizeObserver) sizeObserver.disconnect(); if (mutationObserver) mutationObserver.disconnect();
      doc.removeEventListener('click', schedule, true); doc.removeEventListener('load', schedule, true);
      document.removeEventListener('pointerdown', outside);
    } };
    resize();
  }
  function installNativeForm(frame, doc, firstLoad) {
    quietEditorFrame(frame.contentWindow, doc);
    var found = postForm(doc); if (!found) return false;
    var form = found.form, textarea = found.textarea;
    var action = localUrl(form.getAttribute('action') || frame.contentWindow.location.href, frame.contentWindow.location.href);
    if (!action || !/\/messages\.php$/i.test(action.pathname)) return false;
    clearTimeout(frame._rpmcLoadTimer);
    if (form._rpmcInstalled) { frameBusy(frame, false); fitFrame(frame, doc); return true; }
    form._rpmcInstalled = true; form.classList.add('rpmc-native-form');
    form.method = 'post'; form.target = ensureSendFrame().name;
    action.searchParams.set('format', 'json'); form.action = action.href;
    var adapter = editorAdapter(frame, textarea);
    installSendFeedback(form); prepareSubject(form, textarea);
    if (firstLoad) adapter.setDraft(state.draft);
    state.editorReady = true;
    var restore = state.chat && $('.rpmc-restore-draft', state.chat); if (restore) restore.remove();
    isolateEditor(doc, form.closest('#post') || form);
    var error = frameError(doc); setStatus(error || state.notice, !!error);
    // WYSI input may live in a contenteditable element rather than textarea.
    doc.addEventListener('input', scheduleDraftSave, true);
    doc.addEventListener('change', scheduleDraftSave, true);
    doc.addEventListener('keyup', scheduleDraftSave, true);
    doc.addEventListener('focusout', saveDraftNow, true);
    var lastClicked = null;
    form.addEventListener('click', function (event) {
      var button = event.target.closest('input[type="submit"], button[type="submit"], button:not([type])');
      if (!button) return;
      if (state.submitting) { event.preventDefault(); event.stopImmediatePropagation(); return; }
      lastClicked = button;
    }, true);
    form.addEventListener('submit', function (event) {
      if (state.submitting) { event.preventDefault(); event.stopImmediatePropagation(); return; }
      var text;
      try { text = adapter.flushToTextarea(); }
      catch (error) { event.preventDefault(); event.stopImmediatePropagation(); setStatus(error.message, true); return; }
      if (state.draft !== text) { state.draft = text; draftRevision++; }
      var submitter = event.submitter || lastClicked; lastClicked = null;
      prepareSubject(form, textarea);
      var submitAction = localUrl(form.action);
      if (isPreview(submitter)) submitAction.searchParams.delete('format'); else submitAction.searchParams.set('format', 'json');
      form.action = submitAction.href;
      var pending = { id: draftTab + ':' + Date.now() + ':' + (++state.outgoingSequence), started: Date.now(), stage: 'submitting',
        submitEvent: event, button: submitter, kind: isPreview(submitter) ? 'preview' : 'send',
        before: new Set(state.rows.filter(function (e) { return e.direction === 'out'; }).flatMap(sourceIds)),
        maxId: Math.max(0, ...state.rows.flatMap(sourceIds)), draft: text, revision: draftRevision,
        adapter: adapter, form: form, textarea: textarea, editor: frame, nativeIssued: false,
        controls: $$('input, button, textarea, select', form).map(function (e) { return { node: e, disabled: !!e.disabled }; }) };
      state.pending = pending; form._rpmcLastPending = pending; state.submitting = true; state.notice = '';
      setStatus(pending.kind === 'preview' ? '\u041e\u0442\u043a\u0440\u044b\u0432\u0430\u044e \u043f\u0440\u0435\u0434\u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440\u2026' : '\u041e\u0442\u043f\u0440\u0430\u0432\u043b\u044f\u044e\u2026');
      if (pending.kind === 'send') {
        var receiver = ensureSendFrame(); receiver._rpmcRequest = pending; pending.receiver = receiver; form.target = receiver.name;
        if (submitter && submitter.hasAttribute('formtarget')) {
          pending.submitter = submitter; pending.oldTarget = submitter.getAttribute('formtarget'); submitter.setAttribute('formtarget', receiver.name);
        }
        startOutgoing(pending); sendFeedback(pending, true); watchSendResponse(pending);
      } else form.target = '_self';
      form.setAttribute('aria-busy', 'true'); saveDraftNow();
      pending.timer = setTimeout(function () {
        if (pending.stage !== 'submitting') return;
        if (pending.kind === 'send') completeSend(pending, false, '\u041e\u0442\u0432\u0435\u0442 \u0441\u0435\u0440\u0432\u0435\u0440\u0430 \u043d\u0435 \u043f\u043e\u043b\u0443\u0447\u0435\u043d. \u041f\u0440\u043e\u0432\u0435\u0440\u044c \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0443 \u043f\u0435\u0440\u0435\u0434 \u043f\u043e\u0432\u0442\u043e\u0440\u043e\u043c.', true);
        else {
          unlockSubmission(pending, false); state.editorReady = false;
          editorFallback(frame, frame._rpmcInitialUrl, '\u041f\u0440\u0435\u0434\u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440 \u043d\u0435 \u043e\u0442\u0432\u0435\u0442\u0438\u043b. \u0422\u0435\u043a\u0441\u0442 \u0434\u043e\u0441\u0442\u0443\u043f\u0435\u043d \u0432 \u0441\u043e\u0445\u0440\u0430\u043d\u0435\u043d\u043d\u043e\u043c \u0447\u0435\u0440\u043d\u043e\u0432\u0438\u043a\u0435.');
        }
      }, 35000);
      // Browser default action is the ONLY POST; cancellations remain isolated
      // until a real result or the deadline. Never retry an ambiguous POST.
      // Defer past ALL submit listeners; a microtask can run between listeners.
      setTimeout(function () {
        if (state.pending !== pending) return;
        if (event.defaultPrevented) setStatus('\u0428\u0442\u0430\u0442\u043d\u044b\u0439 \u0440\u0435\u0434\u0430\u043a\u0442\u043e\u0440 \u043f\u0435\u0440\u0435\u0445\u0432\u0430\u0442\u0438\u043b \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0443. \u041e\u0436\u0438\u0434\u0430\u044e \u0440\u0435\u0437\u0443\u043b\u044c\u0442\u0430\u0442 \u0438\u043b\u0438 \u043e\u043a\u043e\u043d\u0447\u0430\u043d\u0438\u0435 \u043f\u0440\u043e\u0432\u0435\u0440\u043a\u0438.');
        else { pending.nativeIssued = true; if (pending.kind === 'preview') frameBusy(frame, true); setStatus(pending.kind === 'preview' ? '\u041e\u0442\u043a\u0440\u044b\u0432\u0430\u044e \u043f\u0440\u0435\u0434\u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440\u2026' : '\u041e\u0442\u043f\u0440\u0430\u0432\u043b\u044f\u044e\u2026'); }
      });
    }, true);
    bindDirectSubmission(form, adapter, textarea, frame);
    bindEditorKeys(form, textarea);
    frameBusy(frame, false); fitFrame(frame, doc);
    var queued = state.queuedQuotes.splice(0); queued.forEach(function (q) { insertQuote(q.text, q.author); });
    saveDraftNow(); return true;
  }

  function explicitSendSuccess(doc) {
    var noticeNode = $('#pun-redirect .container', doc) || $('#pun-redirect', doc) || $('#pun-main > .info', doc);
    return /(?:\u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435|\u043f\u0438\u0441\u044c\u043c\u043e)\s+(?:\u0443\u0441\u043f\u0435\u0448\u043d\u043e\s+)?\u043e\u0442\u043f\u0440\u0430\u0432\u043b\u0435\u043d\u043e|message\s+(?:has been\s+|successfully\s+)?sent/i.test(
      normalizeText(noticeNode && noticeNode.textContent));
  }
  function unlockSubmission(pending, retainReceiver) {
    clearTimeout(pending.timer); clearTimeout(pending.responseTimer);
    if (pending.receiver && state.sendFrame === pending.receiver) state.sendFrame = null;
    if (!state.pending || state.pending === pending) {
      sendFeedback(pending, false);
      (pending.controls || []).forEach(function (item) { if (item && item.node) item.node.disabled = !!item.disabled; });
    }
    if (state.pending === pending) {
      if (pending.form) { pending.form.removeAttribute('aria-busy'); pending.form.target = ensureSendFrame().name; }
      if (pending.submitter) pending.submitter.setAttribute('formtarget', pending.oldTarget);
      state.pending = null; state.submitting = false;
    }
    if (pending.receiver) {
      if (state.sendFrame === pending.receiver) state.sendFrame = null;
      if (retainReceiver) {
        // The old receiver is never reused. A late load belongs to this operation.
        pending.cleanupTimer = setTimeout(function () { pending.receiver._rpmcRequest = null; pending.receiver.remove(); }, 300000);
      } else {
        clearTimeout(pending.cleanupTimer); pending.receiver._rpmcRequest = null; pending.receiver.remove();
      }
    }
    schedulePoll();
  }

  function completeSend(pending, success, message, uncertain) {
    if (!pending || /^(confirmed|rejected)$/.test(pending.stage)) return;
    var active = state.pending === pending;
    captureDraft();
    pending.stage = success ? 'confirmed' : uncertain ? 'unknown' : 'rejected';
    if (pending.optimistic) {
      pending.optimistic.status = success ? 'sent' : uncertain ? 'unknown' : 'failed';
      pending.optimistic.error = message || ''; pending.optimistic.serverId = pending.serverId || 0;
    }
    // A late result never clears a subsequent editor version or operation.
    if (success && active && draftRevision === pending.revision && state.draft === pending.draft) {
      try { if (pending.adapter.getDraft() === pending.draft) { pending.adapter.setDraft(''); state.draft = ''; draftRevision++; } }
      catch (_) { /* Keep text when editor ownership is uncertain. */ }
    }
    unlockSubmission(pending, !!uncertain); saveDraftNow();
    if (active || !state.pending) {
      state.notice = message || (success ? '\u0421\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435 \u043e\u0442\u043f\u0440\u0430\u0432\u043b\u0435\u043d\u043e.' : '\u041e\u0442\u043f\u0440\u0430\u0432\u043a\u0430 \u043d\u0435 \u043f\u043e\u0434\u0442\u0432\u0435\u0440\u0436\u0434\u0435\u043d\u0430. \u041f\u0440\u043e\u0432\u0435\u0440\u044c \u043f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0443 \u043f\u0435\u0440\u0435\u0434 \u043f\u043e\u0432\u0442\u043e\u0440\u043e\u043c.');
      setStatus(state.notice, !success);
    }
    if (state.chat) renderMessages(state.rows, false);
  }

  function refreshAfterSend(seed) {
    state.lastSendUnindexed = true;
    return refreshHistory(true, { incremental: true, quiet: true, afterSend: true, seed: seed })
      .then(function () { state.lastSendUnindexed = !!state.warnings.length; })
      .catch(function () {
        var info = state.chat && $('.rpmc-history-status', state.chat);
        if (info) info.textContent = '\u041f\u0438\u0441\u044c\u043c\u043e \u043e\u0442\u043f\u0440\u0430\u0432\u043b\u0435\u043d\u043e, \u043d\u043e \u0438\u0441\u0442\u043e\u0440\u0438\u044f \u043f\u043e\u043a\u0430 \u043d\u0435 \u043e\u0431\u043d\u043e\u0432\u0438\u043b\u0430\u0441\u044c. \u041d\u0430\u0436\u043c\u0438 \u21bb.';
      });
  }
  async function handleSendLoad(receiver) {
    var pending = receiver._rpmcRequest;
    if (!pending || pending.kind !== 'send' || pending.checking || /^(confirmed|rejected)$/.test(pending.stage)) return;
    var doc, responseUrl;
    try {
      var href = receiver.contentWindow.location.href; if (href === 'about:blank') return;
      responseUrl = localUrl(href); doc = receiver.contentDocument;
      if (!responseUrl || !doc || !doc.body) return;
    } catch (_) { completeSend(pending, false, '\u041d\u0435 \u0443\u0434\u0430\u043b\u043e\u0441\u044c \u043f\u0440\u043e\u0447\u0438\u0442\u0430\u0442\u044c \u043e\u0442\u0432\u0435\u0442. \u041f\u0440\u043e\u0432\u0435\u0440\u044c \u043f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0443 \u043f\u0435\u0440\u0435\u0434 \u043f\u043e\u0432\u0442\u043e\u0440\u043e\u043c.', true); return; }
    await processSendDocument(pending, doc, responseUrl);
  }
  async function processSendDocument(pending, doc, responseUrl) {
    if (pending.checking || /^(confirmed|rejected)$/.test(pending.stage)) return;
    pending.checking = true;
    try {
      var error = frameError(doc);
      if (error) { completeSend(pending, false, error, false); return; }
      if (isLoginResponse(doc, responseUrl)) { completeSend(pending, false, '\u0421\u0435\u0441\u0441\u0438\u044f \u0437\u0430\u043a\u043e\u043d\u0447\u0438\u043b\u0430\u0441\u044c. \u0412\u043e\u0439\u0434\u0438 \u043d\u0430 \u0444\u043e\u0440\u0443\u043c \u0432 \u0434\u0440\u0443\u0433\u043e\u0439 \u0432\u043a\u043b\u0430\u0434\u043a\u0435.', false); return; }
      var confirmed = explicitSendSuccess(doc), result;
      // Parse JSON if the native forum returns it; do not assume JSON support.
      try { result = JSON.parse(doc.body.textContent); } catch (_) {}
      if (result && result.error) { completeSend(pending, false, '\u0424\u043e\u0440\u0443\u043c \u043e\u0442\u043a\u043b\u043e\u043d\u0438\u043b \u043e\u0442\u043f\u0440\u0430\u0432\u043a\u0443. \u041f\u0440\u043e\u0432\u0435\u0440\u044c \u0442\u0435\u043a\u0441\u0442 \u0438 \u0442\u0435\u043c\u0443.', false); return; }
      var id = result && result.response && Number(result.response.id);
      if (Number.isSafeInteger(id) && id > 0) { pending.serverId = id; confirmed = true; }
      if (confirmed) {
        completeSend(pending, true); refreshAfterSend({ doc: doc, url: responseUrl }); return;
      }
      // Do not infer acknowledgement from history, even for identical texts.
      if (pending.stage !== 'unknown') {
        completeSend(pending, false, '\u0424\u043e\u0440\u0443\u043c \u043d\u0435 \u0432\u0435\u0440\u043d\u0443\u043b \u043e\u0434\u043d\u043e\u0437\u043d\u0430\u0447\u043d\u043e\u0433\u043e \u043f\u043e\u0434\u0442\u0432\u0435\u0440\u0436\u0434\u0435\u043d\u0438\u044f. \u041f\u0440\u043e\u0432\u0435\u0440\u044c \u043f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0443 \u043f\u0435\u0440\u0435\u0434 \u043f\u043e\u0432\u0442\u043e\u0440\u043e\u043c.', true);
        refreshHistory(false, { fresh: true, afterSend: true, boxes: mailboxBoxes().filter(function (box) { return direction(box) === 'out'; }) }).catch(function () {});
      }
    } finally { pending.checking = false; }
  }

  function ensureSendFrame() {
    if (state.sendFrame && state.sendFrame.isConnected) return state.sendFrame;
    var receiver = document.createElement('iframe');
    receiver.name = 'resonance-pm-editor-result-' + currentMessageId + '-' + (++state.sendSequence);
    receiver.setAttribute('data-rpmc-editor', '1');
    // The response is read by the parent; it must not boot another copy of the
    // forum's timers, notification scripts or chat. The visible editor keeps JS.
    receiver.setAttribute('sandbox', 'allow-same-origin allow-forms');
    receiver.setAttribute('aria-hidden', 'true'); receiver.tabIndex = -1; receiver.hidden = true;
    receiver.style.setProperty('display', 'none', 'important');
    receiver.addEventListener('load', function () { handleSendLoad(receiver).catch(function () { var pending = receiver._rpmcRequest; if (pending) completeSend(pending, false, 'Не удалось обработать ответ форума. Проверь отправку.', true); }); });
    $('.rpmc-composer', state.chat).appendChild(receiver); state.sendFrame = receiver;
    return receiver;
  }
  function editorFallback(frame, initialUrl, message) {
    if (frame !== state.frame) return;
    clearTimeout(frame._rpmcLoadTimer); captureDraft(); saveDraftNow();
    var pending = state.pending;
    if (pending && pending.editor === frame) {
      if (pending.kind === 'send') completeSend(pending, false, 'Редактор недоступен; проверь результат отправки.', true);
      else unlockSubmission(pending, false);
    }
    if (frame._rpmcLayout) frame._rpmcLayout.dispose();
    frame.remove(); state.frame = null; state.editorReady = false;
    var host = state.chat && $('.rpmc-composer', state.chat); if (!host) return;
    // Keep outstanding result receivers connected while replacing the composer.
    $$('iframe[data-rpmc-editor]', host).forEach(function (receiver) { document.body.appendChild(receiver); });
    mountSimpleEditorFallback(host, message).then(function () {
      var retry = document.createElement('button'); retry.type = 'button'; retry.className = 'rpmc-restore-draft';
      retry.textContent = 'Вернуть штатный редактор';
      retry.addEventListener('click', function () { if (!state.submitting) { captureDraft(); saveDraftNow(); mountEditor(); } });
      host.appendChild(retry); setStatus(message || 'Используется резервный редактор.', true);
    }).catch(function (error) { setStatus(error.message, true); });
  }

  async function handleFrameLoad(frame, initialUrl) {
    if (frame !== state.frame) return;
    var doc;
    try { doc = frame.contentDocument; if (!doc || !doc.body || frame.contentWindow.location.href === 'about:blank') return; }
    catch (_) { editorFallback(frame, initialUrl, '\u0420\u0435\u0434\u0430\u043a\u0442\u043e\u0440 \u043d\u0435\u0434\u043e\u0441\u0442\u0443\u043f\u0435\u043d. \u041c\u043e\u0436\u043d\u043e \u0432\u043e\u0441\u0441\u0442\u0430\u043d\u043e\u0432\u0438\u0442\u044c \u0435\u0433\u043e \u043a\u043d\u043e\u043f\u043a\u043e\u0439 \u043d\u0438\u0436\u0435.'); return; }
    // DOM-ready and load often report the same document. Never handle it twice.
    if (frame._rpmcHandledDoc === doc) { if (state.editorReady) fitFrame(frame, doc); return; }
    frame._rpmcHandledDoc = doc; clearTimeout(frame._rpmcLoadTimer);
    quietEditorFrame(frame.contentWindow, doc);
    var pending = state.pending, found = postForm(doc), error = frameError(doc);
    if (pending && pending.editor === frame) {
      if (pending.kind === 'send') await processSendDocument(pending, doc, localUrl(frame.contentWindow.location.href));
      else { unlockSubmission(pending, false); state.notice = error || ''; }
    }
    state.editorReady = false;
    if (found && installNativeForm(frame, doc, true)) return;
    if (pending && pending.stage === 'confirmed') { navigateEditor(frame, initialUrl); return; }
    editorFallback(frame, initialUrl, error || '\u0424\u043e\u0440\u0443\u043c \u0432\u0435\u0440\u043d\u0443\u043b \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u0443 \u0431\u0435\u0437 \u0440\u0435\u0434\u0430\u043a\u0442\u043e\u0440\u0430. \u0412\u0435\u0440\u043d\u0438 \u0440\u0435\u0434\u0430\u043a\u0442\u043e\u0440 \u043a\u043d\u043e\u043f\u043a\u043e\u0439 \u043d\u0438\u0436\u0435.');
  }

  function sendSimpleComposer(pending, href) {
    var receiver = document.createElement('iframe');
    receiver.name = 'resonance-pm-editor-result-' + currentMessageId + '-' + (++state.sendSequence);
    receiver.setAttribute('data-rpmc-editor', '1');
    receiver.setAttribute('aria-hidden', 'true');
    receiver.tabIndex = -1;
    receiver.style.cssText = 'position:fixed;left:0;bottom:0;width:2px;height:2px;opacity:0;pointer-events:none;border:0;z-index:-1;';
    pending.receiver = receiver;
    receiver._rpmcRequest = pending;
    var phase = 'load-form', finished = false;

    function cleanupOnly() {
      clearTimeout(pending.timer);
      receiver.removeEventListener('load', onLoad);
    }
    function finish(success, message, uncertain, seed) {
      if (finished) return;
      finished = !uncertain;
      if (!uncertain) cleanupOnly();
      completeSend(pending, !!success, message || '', !!uncertain);
      if (success) refreshAfterSend(seed || null);
    }
    function failBeforePost(message) {
      finish(false, message || 'Не удалось открыть штатную форму отправки.', false);
    }
    function onLoad() {
      if (finished || /^(confirmed|rejected)$/.test(pending.stage)) return;
      var doc, url;
      try {
        if (!receiver.contentWindow || receiver.contentWindow.location.href === 'about:blank') return;
        doc = receiver.contentDocument;
        url = localUrl(receiver.contentWindow.location.href);
        if (!doc || !doc.body || !url) return;
      } catch (_) {
        if (phase === 'submitted') finish(false, 'Ответ сервера не удалось прочитать. Проверь переписку перед повтором.', true);
        else failBeforePost('Не удалось открыть штатную форму отправки.');
        return;
      }

      if (phase === 'load-form') {
        if (isLoginResponse(doc, url)) { failBeforePost('Сессия закончилась. Войди на форум и повтори отправку.'); return; }
        var found = postForm(doc);
        if (!found) {
          var initialError = frameError(doc);
          failBeforePost(initialError || 'Форум открыл страницу без формы отправки.');
          return;
        }
        var form = found.form, textarea = found.textarea;
        try {
          var username = form.elements && form.elements.req_username;
          if (username) username.value = state.partner && state.partner.name || username.value || '';
          var subject = form.querySelector('input[name="req_subject"], input[name="subject"], input[name="req_title"]');
          if (subject) {
            var value = state.subject || cleanSubject(subject.value) || 'Переписка';
            var limit = Number(subject.getAttribute('maxlength'));
            subject.value = limit > 0 ? value.slice(0, limit) : value;
          }
          editorAdapter(receiver, textarea).setDraft(pending.draft);
          editorAdapter(receiver, textarea).flushToTextarea();
          textarea.dispatchEvent(new receiver.contentWindow.Event('input', { bubbles: true }));
          var submit = $$('input[type="submit"], button[type="submit"], button:not([type])', form).find(function (button) { return !isPreview(button); });
          if (!submit) { failBeforePost('В штатной форме не найдена кнопка отправки.'); return; }
          var action = localUrl(form.getAttribute('action') || url.href, url.href);
          if (!action || !/\/messages\.php$/i.test(action.pathname)) { failBeforePost('Некорректный адрес отправки.'); return; }
          action.searchParams.set('format', 'json');
          form.setAttribute('action', action.href);
          form.target = '_self';
          phase = 'submitted';
          pending.nativeIssued = true;
          submit.click();
        } catch (error) {
          if (phase === 'submitted') finish(false, 'Результат отправки неизвестен. Проверь переписку перед повтором.', true);
          else failBeforePost(error.message || 'Не удалось подготовить штатную форму отправки.');
        }
        return;
      }

      processSendDocument(pending, doc, url).then(function () {
        if (pending.stage !== 'unknown') { finished = true; cleanupOnly(); }
      }).catch(function () { finish(false, 'Не удалось обработать ответ форума. Проверь отправку.', true); });
    }

    receiver.addEventListener('load', onLoad);
    document.body.appendChild(receiver);
    pending.timer = setTimeout(function () {
      if (finished || pending.stage !== 'submitting') return;
      if (phase === 'submitted') finish(false, 'Ответ сервера не получен. Проверь переписку перед повтором.', true);
      else failBeforePost('Штатная форма отправки не загрузилась.');
    }, 35000);
    try { receiver.src = href; }
    catch (error) { failBeforePost(error.message || 'Не удалось открыть штатную форму отправки.'); }
  }

  async function fetchNativeEditorForm(href) {
    var u = localUrl(href);
    if (!u) throw new Error('Не удалось определить адрес штатного редактора.');
    u.searchParams.delete('rpmc_mode');
    u.searchParams.delete('rpmc_native');
    u.searchParams.set('_rpmc_editor', '14.4');
    var controller = typeof AbortController !== 'undefined' ? new AbortController() : null;
    var timer = controller ? setTimeout(function () { controller.abort(); }, 15000) : null;
    var response;
    try {
      response = await fetch(u.href, { credentials: 'same-origin', cache: 'no-store',
        headers: { 'X-Requested-With': 'XMLHttpRequest' }, signal: controller ? controller.signal : undefined });
    if (!response || !response.ok) throw new Error('Не удалось загрузить штатный редактор: HTTP ' + (response ? response.status : '?') + '.');
    var html = decodeHtml(await response.arrayBuffer(), response.headers.get('content-type'), document.characterSet);
    var doc = new DOMParser().parseFromString(html, 'text/html');
    var form = doc.querySelector('form#post') || Array.from(doc.querySelectorAll('form')).find(function (candidate) {
      return !!candidate.querySelector('textarea#main-reply, textarea[name="req_message"]');
    });
    if (!form) throw new Error('В штатной странице не найдена форма редактора.');
    var textarea = form.querySelector('textarea#main-reply, textarea[name="req_message"]');
    if (!textarea) throw new Error('В штатной форме не найдено поле сообщения.');
    return { form: form, sourceUrl: u.href };
    } finally { if (timer) clearTimeout(timer); }
  }

  function hideInlineField(field, textarea) {
    if (!field) return;
    var holder = field.closest('label, p, .inputfield, .txtfield, .field, .sf-set, .df-set, li, tr');
    if (holder && !holder.contains(textarea) && !holder.querySelector('#form-buttons, .form-buttons, [contenteditable]') &&
        !$$('input:not([type="hidden"]), button, select, textarea', holder).some(function (control) { return control !== field; })) {
      holder.classList.add('rpmc-inline-hidden-field');
      holder.style.setProperty('display', 'none', 'important');
    } else field.classList.add('rpmc-inline-hidden-field');
  }

  function installInlineNativeForm(form, textarea, href) {
    if (!form || !textarea) return false;
    if (form._rpmcInstalled) return true; form._rpmcInstalled = true;
    form.classList.add('rpmc-native-inline', 'rpmc-native-form');
    form.id = 'post';
    var inlineAction = localUrl(form.getAttribute('action') || href, href);
    if (!inlineAction || !/\/messages\.php$/i.test(inlineAction.pathname)) return false;
    inlineAction.searchParams.set('format', 'json'); form.action = inlineAction.href;
    form.target = ensureSendFrame().name;
    form.method = 'post';
    form.enctype = form.enctype || 'multipart/form-data';
    $$('script', form).forEach(function (node) { node.remove(); });
    $$('*', form).forEach(function (node) { Array.from(node.attributes).forEach(function (attr) { if (/^on/i.test(attr.name)) node.removeAttribute(attr.name); }); });
    form.removeAttribute('hidden'); form.removeAttribute('aria-hidden'); form.removeAttribute('onsubmit');
    textarea.id = 'main-reply';
    textarea.setAttribute('name', textarea.getAttribute('name') || 'req_message');
    textarea.setAttribute('placeholder', 'Написать сообщение…');
    textarea.classList.add('rpmc-inline-textarea');
    var username = form.querySelector('input[name="req_username"]');
    if (username && state.partner) username.value = state.partner.name || username.value || '';
    hideInlineField(username, textarea);
    prepareSubject(form, textarea);
    

    // The native form may contain its own outer legend/title; keep editor controls and attachments only.
    var legend = form.querySelector('fieldset > legend'); if (legend) legend.classList.add('rpmc-inline-hidden-field');
    installSendFeedback(form);
    var adapter = editorAdapter(window, textarea);
    try { adapter.setDraft(state.draft || ''); } catch (_) { textarea.value = state.draft || ''; }
    state.frame = null; state.editorReady = true; state.submitting = false;

    form.addEventListener('input', scheduleDraftSave, true);
    form.addEventListener('change', scheduleDraftSave, true);
    form.addEventListener('focusout', saveDraftNow, true);

    var lastClicked = null;
    form.addEventListener('click', function (event) {
      var button = event.target.closest('input[type="submit"], button[type="submit"], button:not([type])');
      if (!button || !form.contains(button)) return;
      if (state.submitting) { event.preventDefault(); event.stopImmediatePropagation(); return; }
      lastClicked = button;
    }, true);

    form.addEventListener('submit', function (event) {
      if (state.submitting) { event.preventDefault(); event.stopImmediatePropagation(); return; }
      var text;
      try { text = adapter.flushToTextarea(); }
      catch (error) { event.preventDefault(); event.stopImmediatePropagation(); setStatus(error.message, true); return; }
      if (!String(text || '').trim()) { event.preventDefault(); event.stopImmediatePropagation(); setStatus('Напиши сообщение перед отправкой.', true); return; }
      if (state.draft !== text) { state.draft = text; draftRevision++; }
      var submitter = event.submitter || lastClicked; lastClicked = null;
      prepareSubject(form, textarea);
      var preview = isPreview(submitter);
      // Preview inside an inline transplanted form would replace the whole chat page.
      // Keep it safe: do not POST or navigate; the ordinary PM view remains available for preview.
      if (preview) {
        event.preventDefault(); event.stopImmediatePropagation();
        setStatus('Предпросмотр не отправляет сообщение. Для штатного предпросмотра открой «обычные ЛС».');
        saveDraftNow(); return;
      }
      var pending = { id: draftTab + ':' + Date.now() + ':' + (++state.outgoingSequence), started: Date.now(), stage: 'submitting',
        submitEvent: event, button: submitter, kind: 'send',
        before: new Set(state.rows.filter(function (e) { return e.direction === 'out'; }).flatMap(sourceIds)),
        maxId: Math.max(0, ...state.rows.flatMap(sourceIds)), draft: text, revision: draftRevision,
        adapter: adapter, form: form, textarea: textarea, editor: null, nativeIssued: false,
        controls: $$('input, button, textarea, select', form).map(function (e) { return { node: e, disabled: !!e.disabled }; }) };
      state.pending = pending; form._rpmcLastPending = pending; state.submitting = true; state.notice = '';
      var receiver = ensureSendFrame(); receiver._rpmcRequest = pending; pending.receiver = receiver;
      form.target = receiver.name;
      if (submitter && submitter.hasAttribute('formtarget')) {
        pending.submitter = submitter; pending.oldTarget = submitter.getAttribute('formtarget'); submitter.setAttribute('formtarget', receiver.name);
      }
      startOutgoing(pending); sendFeedback(pending, true); watchSendResponse(pending);
      form.setAttribute('aria-busy', 'true'); saveDraftNow(); setStatus('Отправляю…');
      pending.timer = setTimeout(function () {
        if (pending.stage !== 'submitting') return;
        completeSend(pending, false, 'Ответ сервера не получен. Проверь переписку перед повтором.', true);
      }, 35000);
      // The browser/native editor performs the single POST. We never call submit/requestSubmit here.
      setTimeout(function () {
        if (state.pending !== pending) return;
        if (event.defaultPrevented) setStatus('Штатный редактор перехватил отправку. Ожидаю результат; повторно не отправляй.');
        else { pending.nativeIssued = true; setStatus('Отправляю…'); }
      }, 0);
    }, true);

    bindDirectSubmission(form, adapter, textarea, null);
    bindEditorKeys(form, textarea);
    var queued = state.queuedQuotes.splice(0);
    queued.forEach(function (q) { try { adapter.insertQuote(quoteCode(q.text, q.author)); } catch (_) {} });
    saveDraftNow(); setStatus(state.notice || '');
    return true;
  }

  async function mountSimpleEditorFallback(host, message) {
    host.innerHTML = '';
    var form = document.createElement('form');
    form.className = 'rpmc-simple-compose'; form.setAttribute('autocomplete', 'off');
    form.innerHTML = '<div class="rpmc-inline-editor-error">' + esc(message || 'Штатный редактор не загрузился.') + '</div>' +
      '<textarea id="main-reply" name="req_message" class="rpmc-simple-textarea" rows="4" placeholder="Написать сообщение…" aria-label="Сообщение"></textarea>' +
      '<div class="rpmc-simple-actions"><span class="rpmc-simple-hint">BBCode поддерживается</span><button type="submit" class="rpmc-simple-send">Отправить</button></div>';
    host.appendChild(form);
    var textarea = $('.rpmc-simple-textarea', form), send = $('.rpmc-simple-send', form);
    textarea.value = state.draft || ''; state.frame = null; state.editorReady = true;
    addFallbackToolbar(form, textarea);
    installSendFeedback(form);
    textarea.addEventListener('input', function () {
      var value = String(textarea.value || ''); if (state.draft !== value) { state.draft = value; draftRevision++; } scheduleDraftSave();
    });
    textarea.addEventListener('change', scheduleDraftSave); textarea.addEventListener('blur', saveDraftNow);
    textarea.addEventListener('keydown', function (event) {
      if ((event.ctrlKey || event.metaKey) && event.key === 'Enter') { event.preventDefault(); if (form.requestSubmit) form.requestSubmit(send); else send.click(); }
    });
    form.addEventListener('submit', async function (event) {
      event.preventDefault(); if (state.submitting) return;
      var text = String(textarea.value || ''); if (!text.trim()) { setStatus('Напиши сообщение перед отправкой.', true); return; }
      if (state.draft !== text) { state.draft = text; draftRevision++; }
      state.submitting = true; send.disabled = true;
      var sendHref; try { sendHref = await composeHref(); } catch (error) { state.submitting = false; send.disabled = false; setStatus(error.message || 'Не удалось определить адрес отправки.', true); return; }
      send.disabled = false;
      var pending = { id: draftTab + ':' + Date.now() + ':' + (++state.outgoingSequence), started: Date.now(), stage: 'submitting', kind: 'send',
        before: new Set(state.rows.filter(function (e) { return e.direction === 'out'; }).flatMap(sourceIds)), maxId: Math.max(0, ...state.rows.flatMap(sourceIds)),
        draft: text, revision: draftRevision, adapter: simpleEditorAdapter(textarea), form: form, textarea: textarea, nativeIssued: false,
        controls: $$('button, textarea, input, select', form).map(function (e) { return { node: e, disabled: !!e.disabled }; }) };
      state.pending = pending; state.submitting = true; state.notice = ''; startOutgoing(pending); sendFeedback(pending, true);
      pending.controls.forEach(function (item) { item.node.disabled = true; }); setStatus('Отправляю…'); saveDraftNow(); sendSimpleComposer(pending, sendHref);
    });
    var queued = state.queuedQuotes.splice(0); queued.forEach(function (q) { simpleEditorAdapter(textarea).insertQuote(quoteCode(q.text, q.author)); });
    saveDraftNow();
  }

  async function mountEditor() {
    var host = state.chat && $('.rpmc-composer', state.chat); if (!host) return;
    var generation = ++state.frameGeneration;
    captureDraft();
    if (state.frame && state.frame._rpmcLayout) state.frame._rpmcLayout.dispose();
    host.innerHTML = '<div class="rpmc-editor-loading">Загружаю штатную панель редактора…</div>';
    state.editorReady = false;
    try {
      var href = await composeHref();
      if (!host.isConnected || generation !== state.frameGeneration) return;
      var frame = document.createElement('iframe');
      frame.name = 'resonance-pm-editor-' + draftTab + '-' + generation;
      frame.setAttribute('data-rpmc-editor', '1'); frame.className = 'rpmc-editor-frame';
      frame.title = 'Редактор личного сообщения'; frame.setAttribute('scrolling', 'auto');
      frame.style.cssText = 'width:100%;border:0;min-height:340px;display:block;';
      state.frame = frame; host.replaceChildren(frame); frameBusy(frame, true);
      frame.addEventListener('load', function () {
        handleFrameLoad(frame, href).catch(function (error) {
          if (frame === state.frame) editorFallback(frame, href, error.message || 'Не удалось подключить редактор.');
        });
      });
      await navigateEditor(frame, href);
    } catch (error) {
      if (host.isConnected && generation === state.frameGeneration) {
        await mountSimpleEditorFallback(host, error.message || 'Не удалось открыть штатный редактор.');
        setStatus(error.message || 'Используется резервный редактор.', true);
      }
    }
  }

  function injectStyles() {
    var oldStyle = $('#resonance-pm-chat-style');
    if (oldStyle) oldStyle.remove();

    var style = document.createElement('style');
    style.id = 'resonance-pm-chat-style';
    style.textContent = `
      .rpmc-history-controls { display: flex; flex-wrap: wrap; gap: 6px; padding: 8px 12px; }
      .rpmc-history-controls button, .rpmc-operation-action { cursor: pointer; padding: 5px 9px; border: 1px solid #315f6233; border-radius: 5px; color: #315f62; background: #ffffff80; font: inherit; }
      .rpmc-draft-state { color: #953a28; padding: 0 12px; font-size: 12px; }
      .rpmc-draft-state:empty { display: none; }
      #resonance-pm-chat {
        --rpmc-accent: #315f62;
        --rpmc-accent-dark: #274e50;
        --rpmc-bg: #d7d7d7;
        --rpmc-panel: rgba(255,255,255,.36);
        --rpmc-in: #f1f1f1;
        --rpmc-out: #3d6b6d;
        --rpmc-text: #2f2f2f;
        --rpmc-muted: #7b7b7b;
        width: 100%;
        margin: 10px 0 18px;
        color: var(--rpmc-text);
        font-family: inherit;
        box-sizing: border-box;
      }
      #resonance-pm-chat *,
      #resonance-pm-chat *:before,
      #resonance-pm-chat *:after { box-sizing: border-box; }

      .rpmc-shell {
        overflow: hidden;
        border: 1px solid rgba(0,0,0,.08);
        border-radius: 7px;
        background: var(--rpmc-bg);
        box-shadow: 0 2px 10px rgba(0,0,0,.04);
      }

      .rpmc-head {
        min-height: 62px;
        padding: 9px 12px;
        display: flex;
        align-items: center;
        gap: 10px;
        background: var(--rpmc-panel);
        border-bottom: 1px solid rgba(0,0,0,.07);
      }

      .rpmc-back {
        width: 31px;
        height: 31px;
        flex: 0 0 31px;
        display: inline-flex;
        align-items: center;
        justify-content: center;
        border-radius: 4px;
        background: var(--rpmc-accent);
        color: #fff !important;
        text-decoration: none !important;
        font-size: 18px;
        line-height: 1;
      }
      .rpmc-back:hover { background: var(--rpmc-accent-dark); }

      .rpmc-head-avatar {
        width: 39px;
        height: 39px;
        flex: 0 0 39px;
        overflow: hidden;
        border-radius: 50%;
        background: rgba(0,0,0,.08);
      }
      .rpmc-head-avatar img,
      .rpmc-mini-avatar img {
        width: 100%;
        height: 100%;
        display: block;
        object-fit: cover;
      }
      .rpmc-avatar-placeholder {
        width: 100%;
        height: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
        background: var(--rpmc-accent);
        color: #fff;
        font-weight: 700;
      }

      .rpmc-person { min-width: 0; flex: 1 1 auto; }
      .rpmc-name {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
        font-size: 12px;
        font-weight: 700;
        text-transform: uppercase;
        letter-spacing: .025em;
      }
      .rpmc-profile {
        display: inline-block;
        margin-top: 1px;
        color: var(--rpmc-accent) !important;
        text-decoration: none !important;
        font-size: 9px;
      }
      .rpmc-count {
        flex: 0 0 auto;
        color: var(--rpmc-muted);
        font-size: 9px;
        white-space: nowrap;
      }

      .rpmc-loading,
      .rpmc-error {
        padding: 38px 20px;
        text-align: center;
        color: var(--rpmc-muted);
        font-size: 11px;
      }

      .rpmc-messages {
        min-height: 260px;
        max-height: 620px;
        overflow-y: auto;
        padding: 18px 14px 20px;
        scroll-behavior: auto;
        overflow-anchor: none;
      }

      .rpmc-row {
        width: 100%;
        display: flex;
        align-items: flex-end;
        gap: 7px;
        margin: 0 0 9px;
      }
      .rpmc-row.in { justify-content: flex-start; }
      .rpmc-row.out { justify-content: flex-end; }

      .rpmc-mini-avatar {
        width: 27px;
        height: 27px;
        flex: 0 0 27px;
        overflow: hidden;
        border-radius: 50%;
        background: rgba(0,0,0,.08);
        margin-bottom: 14px;
      }

      .rpmc-bubble-wrap { max-width: min(74%, 610px); }
      .rpmc-row.out .rpmc-bubble-wrap {
        display: flex;
        flex-direction: column;
        align-items: flex-end;
      }

      .rpmc-bubble {
        display: inline-block;
        max-width: 100%;
        padding: 9px 11px;
        border-radius: 5px 14px 14px 14px;
        background: var(--rpmc-in);
        color: var(--rpmc-text);
        box-shadow: 0 1px 2px rgba(0,0,0,.07);
        overflow-wrap: anywhere;
        line-height: 1.42;
      }
      .rpmc-row.out .rpmc-bubble {
        border-radius: 14px 5px 14px 14px;
        background: var(--rpmc-out);
        color: #fff;
      }

      .rpmc-bubble p,
      .rpmc-bubble div { max-width: 100%; }
      .rpmc-bubble p:first-child { margin-top: 0 !important; }
      .rpmc-bubble p:last-child { margin-bottom: 0 !important; padding-bottom: 0 !important; }
      .rpmc-bubble img { max-width: 100% !important; height: auto !important; }
      .rpmc-bubble table { max-width: 100% !important; width: auto !important; }

      .rpmc-bubble .quote-box,
      .rpmc-bubble blockquote,
      .rpmc-bubble .quote {
        margin: 0 0 7px !important;
        padding: 7px 9px !important;
        border: 0 !important;
        border-left: 2px solid rgba(49,95,98,.45) !important;
        border-radius: 4px !important;
        background: rgba(0,0,0,.055) !important;
        color: inherit !important;
        max-height: 120px;
        overflow: auto !important;
      }
      .rpmc-row.out .rpmc-bubble .quote-box,
      .rpmc-row.out .rpmc-bubble blockquote,
      .rpmc-row.out .rpmc-bubble .quote {
        border-left-color: rgba(255,255,255,.5) !important;
        background: rgba(255,255,255,.10) !important;
      }

      .rpmc-meta {
        margin: 3px 5px 0;
        color: var(--rpmc-muted);
        font-size: 8px;
        line-height: 1.25;
      }
      .rpmc-row.out .rpmc-meta { text-align: right; }
      .rpmc-open-original {
        color: inherit !important;
        opacity: .62;
        text-decoration: none !important;
      }
      .rpmc-open-original:hover { opacity: 1; }

      .rpmc-composer {
        padding: 10px 11px;
        display: flex;
        align-items: flex-end;
        gap: 8px;
        border-top: 1px solid rgba(0,0,0,.07);
        background: rgba(255,255,255,.28);
      }
      .rpmc-textarea {
        min-height: 54px;
        max-height: 160px;
        flex: 1 1 auto;
        resize: vertical;
        padding: 9px 10px;
        border: 1px solid rgba(0,0,0,.12);
        border-radius: 6px;
        outline: 0;
        background: rgba(255,255,255,.75);
        color: var(--rpmc-text);
        font: inherit;
        line-height: 1.4;
      }
      .rpmc-textarea:focus {
        border-color: rgba(49,95,98,.58);
        box-shadow: 0 0 0 2px rgba(49,95,98,.08);
      }
      .rpmc-send {
        min-width: 92px;
        min-height: 34px;
        padding: 7px 13px;
        border: 0;
        border-radius: 5px;
        background: var(--rpmc-accent);
        color: #fff;
        font: inherit;
        font-size: 9px;
        font-weight: 700;
        text-transform: uppercase;
        cursor: pointer;
      }
      .rpmc-send:hover { background: var(--rpmc-accent-dark); }
      .rpmc-send:disabled { opacity: .55; cursor: default; }

      #resonance-pm-chat .rpmc-simple-compose { margin: 0; padding: 0; width: 100%; }
      #resonance-pm-chat .rpmc-simple-textarea {
        display: block; box-sizing: border-box; width: 100%; min-height: 96px; max-height: 280px; resize: vertical;
        padding: 11px 12px; border: 1px solid rgba(49,95,98,.22); border-radius: 9px; outline: 0;
        background: rgba(255,255,255,.82); color: var(--rpmc-text); font: inherit; font-size: 12px; line-height: 1.5;
      }
      #resonance-pm-chat .rpmc-simple-textarea:focus { border-color: rgba(49,95,98,.58); box-shadow: 0 0 0 2px rgba(49,95,98,.08); }
      #resonance-pm-chat .rpmc-simple-actions { display: flex; align-items: center; justify-content: flex-end; gap: 10px; padding-top: 8px; }
      #resonance-pm-chat .rpmc-simple-hint { margin-right: auto; color: var(--rpmc-muted); font-size: 10px; }
      #resonance-pm-chat .rpmc-simple-send { min-width: 108px; min-height: 34px; padding: 8px 14px; border: 0; border-radius: 7px; background: var(--rpmc-accent); color: #fff; font: inherit; font-weight: 700; cursor: pointer; }
      #resonance-pm-chat .rpmc-simple-send:hover { background: var(--rpmc-accent-dark); }
      #resonance-pm-chat .rpmc-simple-send:disabled, #resonance-pm-chat .rpmc-simple-textarea:disabled { opacity: .58; cursor: wait; }

      .rpmc-status {
        min-height: 14px;
        padding: 0 12px 7px;
        background: rgba(255,255,255,.28);
        color: var(--rpmc-muted);
        font-size: 8px;
        text-align: right;
      }
      .rpmc-status.error { color: #8c3e3e; }

      @media (max-width: 650px) {
        .rpmc-messages { max-height: 70vh; padding: 14px 9px 18px; }
        .rpmc-bubble-wrap { max-width: 84%; }
        .rpmc-count { display: none; }
        .rpmc-mini-avatar { display: none; }
        .rpmc-composer { align-items: stretch; flex-direction: column; }
        .rpmc-send { width: 100%; }
      }

      #resonance-pm-chat { font-size: 12px; line-height: 1.45; }
      #resonance-pm-chat .rpmc-shell { border-radius: 18px; overflow: visible; }
      #resonance-pm-chat .rpmc-head { border-radius: 18px 18px 0 0; }
      #resonance-pm-chat .rpmc-status { border-radius: 0 0 18px 18px; }
      #resonance-pm-chat .rpmc-composer { position: relative; z-index: 3; overflow: visible; }
      #resonance-pm-chat .rpmc-head { padding: 12px 14px; }
      #resonance-pm-chat .rpmc-name { text-transform: none; font-size: 14px; }
      #resonance-pm-chat .rpmc-count, #resonance-pm-chat .rpmc-profile { font-size: 11px; }
      #resonance-pm-chat .rpmc-meta { font-size: 10px; }
      #resonance-pm-chat .rpmc-bubble-wrap { min-width: 0; max-width: min(78%, 690px); }
      #resonance-pm-chat .rpmc-bubble { max-width: 100%; font-size: 12px; }
      #resonance-pm-chat .rpmc-bubble pre { max-width: 100%; overflow: auto; white-space: pre-wrap; overflow-wrap: anywhere; }
      #resonance-pm-chat .rpmc-bubble iframe, #resonance-pm-chat .rpmc-bubble video { max-width: 100%; }
      #resonance-pm-chat .rpmc-bubble code { white-space: pre-wrap; overflow-wrap: anywhere; }
      #resonance-pm-chat .rpmc-row.out .rpmc-bubble a { color: #e6f1e8; }
      #resonance-pm-chat .rpmc-messages { display: flex; flex-direction: column; box-sizing: border-box; height: min(560px, 58vh); min-height: 260px; max-height: none; overscroll-behavior: contain; scrollbar-width: thin; scrollbar-color: #648487 transparent; }
      #resonance-pm-chat .rpmc-messages > .rpmc-row { flex-shrink: 0; }
      #resonance-pm-chat .rpmc-messages > .rpmc-row:first-child { margin-top: auto; }
      #resonance-pm-chat .rpmc-composer { display: block; padding: 8px; }
      #resonance-pm-chat .rpmc-editor-frame { display: block; width: 100%; min-height: 340px; border: 0; }
      #resonance-pm-chat .rpmc-editor-loading { padding: 14px; color: var(--rpmc-muted); font-size: 11px; }
      #resonance-pm-chat .rpmc-inline-editor-error { margin: 0 0 7px; padding: 7px 9px; border-radius: 5px; background: rgba(149,58,40,.08); color: #873c2d; font-size: 10px; }
      #resonance-pm-chat form.rpmc-native-inline { display: block !important; width: 100% !important; max-width: none !important; margin: 0 !important; padding: 0 !important; background: transparent !important; border: 0 !important; box-shadow: none !important; overflow: visible !important; }
      #resonance-pm-chat form.rpmc-native-inline fieldset { min-width: 0 !important; margin: 0 !important; }
      #resonance-pm-chat form.rpmc-native-inline .rpmc-inline-hidden-field { display: none !important; }
      #resonance-pm-chat form.rpmc-native-inline #form-buttons { width: 100% !important; max-width: 100% !important; box-sizing: border-box !important; }
      #resonance-pm-chat form.rpmc-native-inline #form-buttons tr { white-space: normal !important; text-align: left !important; }
      #resonance-pm-chat form.rpmc-native-inline #main-reply, #resonance-pm-chat form.rpmc-native-inline textarea[name="req_message"] { width: 100% !important; min-height: 150px !important; max-width: 100% !important; resize: vertical !important; box-sizing: border-box !important; }
      #resonance-pm-chat form.rpmc-native-inline .formsubmit { display: flex !important; flex-wrap: wrap !important; align-items: center !important; gap: 6px !important; }
      #resonance-pm-chat .rpmc-status { padding: 6px 14px 12px; font-size: 11px; text-align: left; }
      #resonance-pm-chat .rpmc-history-status { padding: 8px 14px 0; font-size: 11px; color: #704c32; }
      #resonance-pm-chat .rpmc-history-status:empty { display: none; }
      #resonance-pm-chat .rpmc-refresh { border: 0; background: transparent; color: #315f62; cursor: pointer; font: 24px/1 sans-serif; padding: 4px; }
      #resonance-pm-chat .rpmc-native { font-size: 10px; white-space: nowrap; color: #315f62; }
      #resonance-pm-chat .rpmc-restore-draft { margin: 8px; padding: 8px 12px; border-radius: 6px; border: 0; background: #315f62; color: white; cursor: pointer; }
      #resonance-pm-chat button:focus-visible, #resonance-pm-chat a:focus-visible { outline: 2px solid #315f62; outline-offset: 3px; }
      #resonance-pm-chat .rpmc-bubble { font-weight: 400; }
      #resonance-pm-chat .rpmc-quote {
        display: block; margin: 0 0 8px; padding: 0; border: 0; border-left: 2px solid #719391;
        border-radius: 0 7px 7px 0; background: rgba(49,95,98,.065); color: inherit;
        max-height: none; height: auto; overflow: visible; text-align: left;
      }
      #resonance-pm-chat .rpmc-row.out .rpmc-quote { background: rgba(255,255,255,.09); border-left-color: #acd0c6; }
      #resonance-pm-chat .rpmc-quote-summary { display: block; list-style: none; position: relative; padding: 7px 25px 7px 9px; cursor: pointer; }
      #resonance-pm-chat .rpmc-quote-summary::-webkit-details-marker { display: none; }
      #resonance-pm-chat .rpmc-quote-summary::after { content: '\u2304'; position: absolute; right: 8px; top: 7px; opacity: .7; }
      #resonance-pm-chat .rpmc-quote[open] > .rpmc-quote-summary::after { content: '\u2303'; }
      #resonance-pm-chat .rpmc-quote-author { display: block; margin: 0 0 3px; padding: 0; border: 0; background: transparent; font-size: 10px; font-weight: 600; color: inherit; }
      #resonance-pm-chat .rpmc-quote-preview { display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 2; overflow: hidden; opacity: .82; font-size: 11px; font-weight: 400; }
      #resonance-pm-chat .rpmc-quote[open] > .rpmc-quote-summary .rpmc-quote-preview { display: none; }
      #resonance-pm-chat .rpmc-quote-content { padding: 0 9px 8px; font-size: 11px; max-height: none; overflow: visible; }
      #resonance-pm-chat .rpmc-quote-content .rpmc-quote { margin-top: 3px; background: transparent; }
      #resonance-pm-chat .rpmc-quote-button { margin: 0 0 0 9px; padding: 0; border: 0; background: transparent; box-shadow: none; color: #476f70; font: inherit; font-size: 10px; cursor: pointer; text-transform: none; }
      #resonance-pm-chat .rpmc-quote-button:hover { text-decoration: underline; }
      #resonance-pm-chat .rpmc-selection-quote { position: fixed; z-index: 2147483000; padding: 7px 11px; border: 0; border-radius: 7px; background: #315f62; color: #fff; box-shadow: 0 3px 12px #0002; font: 12px/1.4 sans-serif; cursor: pointer; }
      #resonance-pm-chat .rpmc-selection-quote[hidden] { display: none !important; }
      #resonance-pm-chat .rpmc-status { font-size: 12px; font-weight: 500; line-height: 1.5; color: #315f62; }
      #resonance-pm-chat .rpmc-status:empty { display: none; }
      #resonance-pm-chat .rpmc-status.is-sending {
        display: flex; align-items: center; gap: 9px; padding: 11px 15px; background: #c4d8d4; font-weight: 600;
      }
      #resonance-pm-chat .rpmc-status.error { padding: 11px 15px; color: #8b3838; background: #eedbda; }
      #resonance-pm-chat .rpmc-pending-text { white-space: pre-wrap; opacity: .86; }
      #resonance-pm-chat .rpmc-delivery-state {
        display: flex; align-items: center; gap: 6px; margin: 6px 3px 3px; font: 600 11px/1.4 sans-serif; color: #315f62;
      }
      #resonance-pm-chat .rpmc-pending-row[data-delivery="sending"] .rpmc-delivery-state::before,
      #resonance-pm-chat .rpmc-status.is-sending::before {
        content: ''; flex: 0 0 12px; height: 12px; border: 2px solid #315f6255; border-top-color: #315f62;
        border-radius: 50%; animation: rpmc-chat-spin .8s linear infinite;
      }
      #resonance-pm-chat .rpmc-pending-row[data-delivery="sent"] .rpmc-delivery-state::before { content: '\u2713'; }
      #resonance-pm-chat .rpmc-pending-row[data-delivery="sent"] .rpmc-pending-text { opacity: 1; }
      #resonance-pm-chat .rpmc-pending-row[data-delivery="failed"] .rpmc-bubble { outline: 2px solid #a96969; }
      #resonance-pm-chat .rpmc-pending-row[data-delivery="failed"] .rpmc-delivery-state { color: #8b3838; }
      #resonance-pm-chat .rpmc-pending-row[data-delivery="unknown"] .rpmc-delivery-state { color: #78522b; }
      #resonance-pm-chat .rpmc-spoiler {
        display: block; margin: 6px 0; border: 1px solid #71939177; border-radius: 8px;
        background: #315f620b; color: inherit; text-align: left; overflow: hidden;
      }
      #resonance-pm-chat .rpmc-spoiler > summary {
        display: block; list-style: none; position: relative; padding: 9px 31px 9px 11px;
        font-weight: 600; cursor: pointer; user-select: none;
      }
      #resonance-pm-chat .rpmc-spoiler > summary::-webkit-details-marker { display: none; }
      #resonance-pm-chat .rpmc-spoiler > summary::after { content: '+'; position: absolute; right: 12px; font-size: 15px; line-height: 1.2; }
      #resonance-pm-chat .rpmc-spoiler[open] > summary::after { content: '\u2212'; }
      #resonance-pm-chat .rpmc-spoiler > summary:focus-visible { outline: 2px solid currentColor; outline-offset: -3px; }
      #resonance-pm-chat .rpmc-spoiler-content { padding: 9px 11px 11px; border-top: 1px solid #71939155; }
      #resonance-pm-chat .out .rpmc-spoiler { background: #ffffff0a; border-color: #ffffff44; }
      #resonance-pm-chat .out .rpmc-spoiler-content { border-top-color: #ffffff33; }
      @keyframes rpmc-chat-spin { to { transform: rotate(360deg); } }
      @media (prefers-reduced-motion: reduce) {
        #resonance-pm-chat .rpmc-pending-row .rpmc-delivery-state::before,
        #resonance-pm-chat .rpmc-status.is-sending::before { animation: none !important; }
      }
      #resonance-pm-chat .rpmc-new-incoming {
        display: block; margin: 0 auto 8px; padding: 6px 12px; border: 0; border-radius: 12px;
        background: #315f62; color: #fff; font: inherit; font-size: 11px; cursor: pointer;
      }
      #resonance-pm-chat .rpmc-new-incoming[hidden] { display: none !important; }
      #resonance-pm-chat .rpmc-messages { overflow-anchor: none; }
      #resonance-pm-modes { display: flex; flex-wrap: wrap; gap: 5px; margin: 12px 0; }
      #resonance-pm-modes a {
        display: inline-block; padding: 8px 14px; border-radius: 9px; text-decoration: none;
        background: rgba(255,255,255,.3); color: #315f62; font-size: 12px; line-height: 1.3;
      }
      #resonance-pm-modes a.is-active { color: #fff; background: #315f62; }
      #resonance-pm-dialogs {
        width: 100%; margin: 12px 0 20px; overflow: hidden; border: 1px solid #0001;
        border-radius: 18px; background: #d7d7d7; color: #303637; font-family: inherit; font-size: 12px; line-height: 1.5;
      }
      #resonance-pm-dialogs, #resonance-pm-dialogs * { box-sizing: border-box; }
      #resonance-pm-dialogs .rpmc-dialog-head {
        display: flex; align-items: center; gap: 14px; padding: 15px 18px;
        background: rgba(255,255,255,.3); border-bottom: 1px solid #0001;
      }
      #resonance-pm-dialogs .rpmc-dialog-head strong { flex: 1; font-size: 16px; }
      #resonance-pm-dialogs .rpmc-new-message { color: #315f62; font-size: 11px; }
      #resonance-pm-dialogs .rpmc-refresh { padding: 2px 6px; border: 0; background: transparent; color: #315f62; font: 24px/1 sans-serif; cursor: pointer; }
      #resonance-pm-dialogs .rpmc-dialog-controls { display: flex; flex-wrap: wrap; align-items: center; gap: 14px; padding: 14px 18px; }
      #resonance-pm-dialogs .rpmc-dialog-filter {
        flex: 1; min-width: 130px; width: auto; padding: 9px 12px; border: 1px solid #0001;
        border-radius: 10px; background: rgba(255,255,255,.6); color: #303637; font: inherit;
      }
      #resonance-pm-dialogs .rpmc-dialog-controls label { display: flex; align-items: center; gap: 5px; cursor: pointer; font-size: 11px; }
      #resonance-pm-dialogs .rpmc-dialog {
        display: flex; align-items: center; gap: 13px; padding: 16px 18px; margin: 0;
        border: 0; border-top: 1px solid #0001; text-decoration: none; color: #303637;
        background: rgba(255,255,255,.17); text-align: left;
      }
      #resonance-pm-dialogs .rpmc-dialog:hover, #resonance-pm-dialogs .rpmc-dialog:focus-visible { background: rgba(255,255,255,.5); }
      #resonance-pm-dialogs .rpmc-dialog.is-unread { background: rgba(244,251,250,.55); }
      #resonance-pm-dialogs .rpmc-dialog-avatar { flex: 0 0 45px; width: 45px; height: 45px; border-radius: 13px; overflow: hidden; }
      #resonance-pm-dialogs .rpmc-dialog-avatar img { width: 100%; height: 100%; object-fit: cover; }
      #resonance-pm-dialogs .rpmc-avatar-placeholder { background: #315f62; color: #fff; font-size: 18px; }
      #resonance-pm-dialogs .rpmc-dialog-body { flex: 1; min-width: 0; }
      #resonance-pm-dialogs .rpmc-dialog-body strong { display: block; font-size: 13px; font-weight: 600; }
      #resonance-pm-dialogs .rpmc-dialog-preview {
        display: block; max-width: 100%; overflow: hidden; text-overflow: ellipsis;
        white-space: nowrap; opacity: .68; font-size: 11px; margin-top: 3px;
      }
      #resonance-pm-dialogs .rpmc-dialog-meta { display: flex; flex-direction: column; align-items: flex-end; gap: 6px; font-size: 10px; }
      #resonance-pm-dialogs .rpmc-dialog-meta time { opacity: .65; }
      #resonance-pm-dialogs .rpmc-unread-badge { min-width: 21px; padding: 1px 7px; text-align: center; border-radius: 10px; background: #315f62; color: #fff; }
      #resonance-pm-dialogs .rpmc-list-status { margin: 0; padding: 0 18px 12px; color: #765638; font-size: 11px; }
      #resonance-pm-dialogs .rpmc-list-status:empty { display: none; }
      #resonance-pm-dialogs .rpmc-dialog-empty { padding: 20px; text-align: center; opacity: .7; }
      @media (max-width: 650px) {
        #resonance-pm-chat .rpmc-head { flex-wrap: wrap; gap: 7px; }
        #resonance-pm-chat .rpmc-bubble-wrap { max-width: 94%; }
        #resonance-pm-chat .rpmc-composer { padding: 0; }
      }
    `;

    document.head.appendChild(style);
  }



  async function main() {
    state.mode = resolveMode();
    injectStyles(); mountModes();
    if (state.mode === 'native' || pageUrl.searchParams.get('action')) return;
    restoreCache();
    if (!currentMessageId) {
      try {
        // The current mailbox is already in the document. Show it, together with
        // the cached index, before starting any background requests.
        var immediate = new Map(state.mailboxRows.map(function (e) { return [e.direction + ':' + e.id, e]; }));
        parseMailboxRows(document, currentBox).forEach(function (e) { immediate.set(e.direction + ':' + e.id, e); });
        state.mailboxRows = Array.from(immediate.values());
        mountDialogues(); renderDialogues(); hideMailboxNodes(); startPolling();
        state.autoHistory = false;
        await refreshDialogues(false, { doc: document, url: pageUrl });
      } catch (_) {
        if (state.list) state.list.remove(); state.list = null; showNative();
      }
      return;
    }
    var current = currentRowFromPage(); if (!current) return;
    try {
      var known = state.mailboxRows.find(function (e) { return e.id === currentMessageId && e.box === currentBox; });
      current = await resolveCompanion(current, known);
      state.current = current;
      if (!state.mailboxRows.some(function (e) { return e.id === current.id && e.direction === current.direction; })) state.mailboxRows.push(current);
      state.subject = cleanSubject(current.subject) || '\u041f\u0435\u0440\u0435\u043f\u0438\u0441\u043a\u0430';
      state.partner = { id: current.partnerId, name: current.partnerName || '\u0421\u043e\u0431\u0435\u0441\u0435\u0434\u043d\u0438\u043a', href: current.partnerHref, avatar: '' };
      restoreDraft();
      var post = nativePost(document), profile = ownProfileLinks($('.post-author', post) || post)[0];
      var avatar = $('.pa-avatar img', post);
      if (profile && linkId(profile) === state.partner.id && avatar) {
        state.partner.avatar = avatar.getAttribute('src'); storeAvatar(state.partner.id, state.partner.avatar);
      }
      // All of this is local work. Even if the server is slow, the cached chat
      // and the already opened letter are visible immediately.
      memory.set(current.direction + ':' + current.id, extractMessage(document));
      state.initializing = true;
      mountChat(state.partner); showCachedHistory(true); hideNative(); scheduleCacheSave();
      await mountEditor();
      state.initializing = false;
      startPolling(); state.autoHistory = false;
      var refresh = refreshHistory(false, { incremental: true, quiet: true });
      hydrateAvatars(state.chat);
      await refresh;
    } catch (error) {
      state.initializing = false; state.autoHistory = false;
      if (state.chat) { state.chat.remove(); state.chat = null; } showNative();
      var notice = document.createElement('p'); notice.className = 'rpmc-fallback-note';
      notice.textContent = '\u0427\u0430\u0442 \u043d\u0435 \u043e\u0442\u043a\u0440\u044b\u043b\u0441\u044f: ' + error.message + '. \u041d\u0438\u0436\u0435 \u043e\u0431\u044b\u0447\u043d\u043e\u0435 \u043f\u0438\u0441\u044c\u043c\u043e.';
      var post = nativePost(document); if (post) post.parentNode.insertBefore(notice, post);
    }
  }
  if (document.readyState === 'loading') document.addEventListener('DOMContentLoaded', function () { main().catch(function (error) { setStatus(error.message, true); showNative(); }); }, { once: true });
  else main().catch(function (error) { setStatus(error.message, true); showNative(); });
})();
