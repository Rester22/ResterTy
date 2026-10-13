# ResterTy
ECN FORM testing
const $ = (s, root=document) => root.querySelector(s);
const $$ = (s, root=document) => [...root.querySelectorAll(s)];
const departments = ["Engineering","QA","Document Control","Production","Warehouse","Purchasing","Sales","Planner","HR"];
const defaultActions = [
["Update customer drawing in server","Engineering"],["Update or create BOM","Engineering"],
["Update or create process drawing","Engineering"],["Update or create PMP","Engineering"],
["Update or create CNC program","Engineering"],["Update or create tooling list","Engineering"],
["Update or create Work Instruction (WI)","Engineering"],["Prepare training plan","Engineering"],
["Update QA check sheet","QA"],["Update CMM program","QA"],["Update checking WI / Do & Don't","QA"],
["Notify supplier about ECN","Purchasing"],["Revise costing","Sales"],["Notify customer for approval","Sales"],
["Obsolete old revision document","Document Control"],["Remove obsolete documents from work area","Document Control"],
["Brief production team on ECN","Production"],["Retrain operators with new WI","Production"],["Others","Other"]
];
let approvals = [];
function escapeHtml(s){return String(s??"").replace(/[&<>"']/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"}[c]));}
function initDepartments(){
  $("#departments").innerHTML=departments.map(d=>`<label class="dept"><input type="checkbox" name="affectedDepartments" value="${d}">${d}</label>`).join("");
}
function addActionRow(action="",dept="Engineering",done=false){
  const body=$("#actionsTable tbody"), n=body.children.length+1, tr=document.createElement("tr");
  tr.innerHTML=`<td><input type="checkbox" aria-label="Action ${n} complete" ${done?"checked":""}></td><td>${n}</td><td><input aria-label="Action item" value="${escapeHtml(action)}"></td><td><select aria-label="Action department">${[...departments,"Other"].filter((x,i,a)=>a.indexOf(x)===i).map(x=>`<option ${x===dept?"selected":""}>${x}</option>`).join("")}</select></td><td><input aria-label="Lead time" placeholder="e.g. 2 days"></td><td><input aria-label="Due date" type="date"></td><td><input aria-label="Person in charge" placeholder="PIC"></td>`;
  body.appendChild(tr);
}
function initInventory(){
 $("#inventoryRows").innerHTML=["Store","Production","Supplier","Customer"].map(x=>`<tr><td>${x}</td>${[0,1,2,3].map(()=>'<td><input type="number" min="0" step="1" value="0"></td>').join("")}<td><input placeholder="Remark"></td></tr>`).join("");
}
const canvas=$("#signatureCanvas"),ctx=canvas.getContext("2d");let drawing=false,hasInk=false;
function resizeCanvas(){
 const rect=canvas.getBoundingClientRect(),ratio=window.devicePixelRatio||1;
 const old=hasInk?canvas.toDataURL():null;
 canvas.width=Math.max(1,Math.floor(rect.width*ratio));canvas.height=Math.max(1,Math.floor(rect.height*ratio));
 ctx.setTransform(ratio,0,0,ratio,0,0);ctx.lineWidth=2;ctx.lineCap="round";ctx.lineJoin="round";ctx.strokeStyle="#10243a";
 if(old){const img=new Image();img.onload=()=>ctx.drawImage(img,0,0,rect.width,rect.height);img.src=old;}
}
function point(e){const r=canvas.getBoundingClientRect();return{x:e.clientX-r.left,y:e.clientY-r.top};}
canvas.addEventListener("pointerdown",e=>{drawing=true;hasInk=true;canvas.setPointerCapture(e.pointerId);const p=point(e);ctx.beginPath();ctx.moveTo(p.x,p.y);});
canvas.addEventListener("pointermove",e=>{if(!drawing)return;const p=point(e);ctx.lineTo(p.x,p.y);ctx.stroke();});
["pointerup","pointercancel","lostpointercapture"].forEach(ev=>canvas.addEventListener(ev,()=>drawing=false));
$("#clearSignature").addEventListener("click",()=>{ctx.clearRect(0,0,canvas.width,canvas.height);hasInk=false;});
function formData(){
 const out={};new FormData($("#ecnForm")).forEach((v,k)=>{if(k==="affectedDepartments"){(out[k]??=[]).push(v)}else if(out[k]!==undefined){out[k]=Array.isArray(out[k])?[...out[k],v]:[out[k],v]}else out[k]=v;});
 out.affectedDepartments= $$('input[name="affectedDepartments"]:checked').map(x=>x.value);
 out.actions=$$("#actionsTable tbody tr").map(tr=>{const i=$$("input",tr),s=$("select",tr);return{done:i[0].checked,item:i[1].value,department:s.value,leadTime:i[2].value,dueDate:i[3].value,pic:i[4].value};});
 out.inventory=$$("#inventoryRows tr").map(tr=>{const cells=$$("input",tr);return{location:tr.cells[0].textContent,total:cells[0].value,useAsIs:cells[1].value,rework:cells[2].value,scrap:cells[3].value,remark:cells[4].value};});
 out.approvals=approvals;return out;
}
function saveDraft(){try{localStorage.setItem("uisEcnDraft",JSON.stringify({form:formData(),signature:hasInk?canvas.toDataURL("image/png"):null,companyLogo:companyLogoData}));$("#approvalStatus").textContent="Draft saved in this browser."; }catch(e){alert("Could not save draft. Browser storage may be full.");}}
function renderApprovals(){
 $("#approvalTable tbody").innerHTML=approvals.map(a=>`<tr><td>${escapeHtml(a.name)}</td><td>${escapeHtml(a.role)}</td><td class="approval-state">${escapeHtml(a.decision)}</td><td>${escapeHtml(a.timestamp)}</td><td>${escapeHtml(a.comment)}</td><td>${a.signature?`<img class="signature-thumb" alt="Saved drawn signature" src="${a.signature}">`:"—"}</td></tr>`).join("");
}
$("#submitApproval").addEventListener("click",()=>{
 const name=$("#reviewerName").value.trim(),role=$("#reviewerRole").value.trim();
 if(!name||!role){$("#approvalStatus").textContent="Enter reviewer name and department / role first.";return;}
 if(!hasInk){$("#approvalStatus").textContent="Please draw your signature before submitting.";return;}
 approvals.push({name,role,decision:$("#reviewerDecision").value,comment:$("#reviewerComment").value.trim(),timestamp:new Date().toLocaleString(),signature:canvas.toDataURL("image/png")});
 renderApprovals();$("#approvalStatus").textContent="Approval record added to this draft. Save/export the record to retain it.";
 $("#reviewerName").value="";$("#reviewerRole").value="";$("#reviewerComment").value="";
 ctx.clearRect(0,0,canvas.width,canvas.height);hasInk=false;
});
function download(name,content,type){const blob=new Blob([content],{type}),url=URL.createObjectURL(blob),a=document.createElement("a");a.href=url;a.download=name;a.click();URL.revokeObjectURL(url);}
$("#saveBtn").addEventListener("click",saveDraft);
$("#loadBtn").addEventListener("click",()=>{
 const raw=localStorage.getItem("uisEcnDraft");if(!raw){alert("No saved draft found in this browser.");return;}
 try{const data=JSON.parse(raw);const f=data.form;
 Object.entries(f).forEach(([k,v])=>{if(["actions","inventory","approvals","affectedDepartments"].includes(k))return;const el=$(`[name="${k}"]`);if(el){if(el.type==="checkbox")el.checked=!!v;else el.value=v;}});
 $$('input[name="affectedDepartments"]').forEach(x=>x.checked=(f.affectedDepartments||[]).includes(x.value));
 $("#actionsTable tbody").innerHTML="";(f.actions||[]).forEach(a=>{addActionRow(a.item,a.department,a.done);const tr=$("#actionsTable tbody").lastElementChild,ins=$$("input",tr);ins[2].value=a.leadTime||"";ins[3].value=a.dueDate||"";ins[4].value=a.pic||"";});
 (f.inventory||[]).forEach((r,i)=>{const ins=$$("input",$("#inventoryRows").rows[i]);[r.total,r.useAsIs,r.rework,r.scrap,r.remark].forEach((v,j)=>{if(ins[j])ins[j].value=v;});});
 approvals=f.approvals||[];renderApprovals();
 if(data.companyLogo){companyLogoData=data.companyLogo;companyLogo.src=companyLogoData;companyLogo.hidden=false;}
 if(data.signature){const img=new Image();img.onload=()=>{ctx.drawImage(img,0,0,canvas.clientWidth,canvas.clientHeight);hasInk=true;};img.src=data.signature;}
 $("#approvalStatus").textContent="Draft loaded.";
 }catch(e){alert("Saved draft could not be read.");}
});
$("#exportBtn").addEventListener("click",()=>{const data=formData();data.exportedAt=new Date().toISOString();download(`${(data.ecnNo||"ECN").replace(/[^a-z0-9_-]/gi,"_")}_record.json`,JSON.stringify(data,null,2),"application/json");});
function printForm(){window.print();}
$("#printBtn").addEventListener("click",printForm);$("#printBottom").addEventListener("click",printForm);
$("#addAction").addEventListener("click",()=>addActionRow());
$("#ecnForm").addEventListener("reset",()=>{setTimeout(()=>{approvals=[];renderApprovals();$("#actionsTable tbody").innerHTML="";defaultActions.forEach(a=>addActionRow(a[0],a[1]));initInventory();ctx.clearRect(0,0,canvas.width,canvas.height);hasInk=false;},0);});

// Company logo upload and preview (stored locally with the browser draft)


initDepartments();defaultActions.forEach(a=>addActionRow(a[0],a[1]));initInventory();
const today=new Date().toISOString().slice(0,10);$('input[name="ecnDate"]').value=today;resizeCanvas();window.addEventListener("resize",resizeCanvas);
