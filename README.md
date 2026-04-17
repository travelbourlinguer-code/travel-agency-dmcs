from fastapi import APIRouter, UploadFile, File, HTTPException, Depends
from sqlalchemy.orm import Session
import os
import shutil
from app.db.database import get_db
from app.models.viaje import Viaje
from app.config import settings
from typing import List

router = APIRouter()

@router.post("/upload/{viaje_id}")
async def subir_archivo(
    viaje_id: int,
    archivo_tipo: str,  # "imagen", "video", "pdf"
    archivo: UploadFile = File(...),
    db: Session = Depends(get_db)
):
    """Subir archivo (imagen, video o PDF) para un viaje"""
    
    viaje = db.query(Viaje).filter(Viaje.id == viaje_id).first()
    if not viaje:
        raise HTTPException(status_code=404, detail="Viaje no encontrado")
    
    # Validar tipo de archivo
    tipo_permitido = {
        "imagen": ["image/jpeg", "image/png", "image/gif"],
        "video": ["video/mp4", "video/mpeg", "video/quicktime"],
        "pdf": ["application/pdf"]
    }
    
    if archivo_tipo not in tipo_permitido:
        raise HTTPException(status_code=400, detail="Tipo de archivo no válido")
    
    if archivo.content_type not in tipo_permitido[archivo_tipo]:
        raise HTTPException(status_code=400, detail=f"Formato no permitido para {archivo_tipo}")
    
    # Crear directorio si no existe
    upload_dir = os.path.join(settings.UPLOAD_DIR, f"viaje_{viaje_id}")
    os.makedirs(upload_dir, exist_ok=True)
    
    # Guardar archivo
    archivo_path = os.path.join(upload_dir, archivo.filename)
    with open(archivo_path, "wb") as buffer:
        shutil.copyfileobj(archivo.file, buffer)
    
    archivo_url = f"/uploads/viaje_{viaje_id}/{archivo.filename}"
    
    # Actualizar viaje según tipo
    if archivo_tipo == "imagen":
        if not viaje.imagenes:
            viaje.imagenes = []
        viaje.imagenes.append(archivo_url)
    
    elif archivo_tipo == "video":
        if not viaje.videos:
            viaje.videos = []
        viaje.videos.append(archivo_url)
    
    elif archivo_tipo == "pdf":
        viaje.pdf_url = archivo_url
    
    db.commit()
    
    return {
        "mensaje": "Archivo subido exitosamente",
        "url": archivo_url,
        "tipo": archivo_tipo
    }

@router.get("/{viaje_id}/multimedia")
async def obtener_multimedia(viaje_id: int, db: Session = Depends(get_db)):
    """Obtener todos los archivos multimedia de un viaje"""
    viaje = db.query(Viaje).filter(Viaje.id == viaje_id).first()
    if not viaje:
        raise HTTPException(status_code=404, detail="Viaje no encontrado")
    
    return {
        "viaje_id": viaje_id,
        "imagenes": viaje.imagenes or [],
        "videos": viaje.videos or [],
        "pdf": viaje.pdf_url
    }
