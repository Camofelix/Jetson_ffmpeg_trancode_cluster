# Jetson_ffmpeg_trancode_cluster

 This project aims to bring together a few things together. 

The inciting goal was to:
lower my power consumptions
learn aarch64 (AKA 64 bit arm)
learn about clustering 
implement platform agnostic hardware accelerated transcoding for ffmpeg

I've also been tracking some ups and downs on the Level1Techs forum in this thread! https://forum.level1techs.com/t/building-a-10-100w-distributed-arm-based-transcode-cluster-i-missed-devember-jobless-january-floundering-february/167831/40


## Components 

### Hardware:

 1. Host server (either dedicated or worskstation PC)
 2. Networking switch (poe?)
 3. Multiple Nvidia Jetson's (ideally NX (agx is the dream)) but probably 2gb nano's or, depending on launch dates of Nvidia roadmap, Jetson Nano Next or Jetson Orin 

Other potential option: 

Turing Machines are launching a V2 of their Turring Pi cluster board, and it will have 4 260 pin slot's, initially deisgned for the Rpi CM4. I've reached out to their questions email adress asking about potential support, but have yet to hear back. Would alow for any mix of Nano (2 GB or 4GB) and Xavier NX + any other Jetson SOM that Nvidia may release. 
 
 ### Software: 
 1. Host 
  1. OS for server- debian based 
  2. Unicorn Transcoder or Kuber Plex 
  3. NFS share (Data itself is accessed over the network on my nas, ZFS and so on)
  4. Custom capture scrip to modify arguments sent from PMS to transcoder 
  5. Plex Media Server
  6. Load balancer  
 
 2. Jetson (aka transcoder node)
  1. Jetpack (4.3?)
  2. client side of UT or KP from 1.2.
  3. custom ffmpeg build
   1. Personal cuda patch (should upstream to newest branch once I'm done) for tonemapping. Currently Reinhard and Hable, but will change to eotf BT.2390 Soon TM
   2. Jcover90 ffmpeg patch to enable the use of the transcode blocks 
   3. Nvidia build of ffmpeg to enable decoding and vf_scale_cuda
  4. Client side of load balancer from 1.6.
  5. (things I forgot will go here)
  
## How do I plan to architect this?

Plan is a 1u to 2u unit(s?) system, probably acrilic box. The system would present an eithernet jack and a power cord. the interior of the device would either have an integrated POE switch or would have an integrated battery bank (would require a AC/DC stepdown to 5v?)  

The distribution of work load via the Unicorn transcoder project [link goes here] or DockerPlex. both achieve the same thing, but the Docker swarm component could be useful for managing nodes

Transcoding of content handled via ffmpeg, but customized/stripped down for only the core/needed components. 

As of now the most optimal way to achieve what we want is to decode in hardware, use cuda to resize and tonemap if needed, then encode the file while sending back to the host. It seems that the NVDEC block doesnt fully support 10 bit content, so may have to use cuda in the case of 10 bit HDR. Isnt really an issue since I'll be moving data to gpu afterwards anyway, and the jetson keeps all decoded/encoded data in GPU memory.
    
## Current State of different parts

cuda based tonemapping filter: Pretty much done. Need to confirm performance but so far so good. Unfortunately CUDA_frames aren't supported. So will have to use a propriatary filter. Thinking vf_scale_jetson and vf_tonemap_jetson for naiming convention. 

Capture Script to modify arguments to HW counterparts: Working well, but haven't done enough edge case testing.

Test "cost" of pulling data across network, and optane on Jetson. Results are great! Reading and writing to/from networked VM network share over NFS on gigabit had a margin of error variance. all local was varience of ~2%, changing to networked produced identical results. 

10 bit/HDR decoding: potential roadblock is that the Nvidia documentation contradicts itself on the topic of 10 bit decoding support. It seems that Gstreamer does support 10 bit by truncating it down to 8 bit; that isnt an issue but will require more in depth testing. In the case where I can't get the decoder to work, Cuda based decoding may be an option. This has the added benifit of taking everything away from the CPU, so I can guarentee more ram to the GPU/encoding blocks.   
