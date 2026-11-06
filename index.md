---
layout: home
title: Zenith 帮助中心
titleTemplate: false

hero:
  name: Zenith 帮助中心
  text: 多云 GPU 算力，从这里开始
  tagline: 从登录、创建开发机到训练、推理、数据流转与项目协作，按真实控制台完成每一步。
  image:
    src: /zenith-hero.svg
    alt: Zenith 多云资源示意图
  actions:
    - theme: brand
      text: 五分钟快速开始
      link: /guide/quick-start
    - theme: alt
      text: 查看资源说明
      link: /resources/overview

features:
  - icon: 🖥️
    title: 创建和管理开发机
    details: 在我的开发机中快速新建，或在开发机管理的向导中选择资源区、规格、镜像、队列与存储。
  - icon: ⌨️
    title: 连接计算环境
    details: 配置 SSH 公钥，使用连接信息或网页终端进入具备访问条件的开发机。
  - icon: ⚡
    title: 训练与推理
    details: 创建训练任务，查看状态、事件和运行节点，并了解受限开放的推理服务能力。
  - icon: 📦
    title: 数据与资产
    details: 按资源区选择已登记的镜像，管理模型元数据，浏览允许访问的存储，并按权限发起文件迁移。
  - icon: 👥
    title: 项目、配额与费用
    details: 理解项目成员、资源标签、配额、月度卡时和估算费用之间的关系。
  - icon: 🧭
    title: 权限与问题排查
    details: 区分平台角色、产品身份和项目角色，并按可行动的检查清单定位常见问题。
---

<div class="home-content">
  <p class="home-eyebrow">CHOOSE YOUR PATH</p>
  <h2>从你要完成的任务开始</h2>
  <p class="home-lead">
    第一次使用建议先完成快速开始。遇到按钮不可用、状态延迟或资源区差异时，再查阅权限、能力与排障说明。
  </p>

  <div class="entry-grid">
    <a class="entry-card" href="./guide/platform-overview.html">
      <span class="entry-card-icon">🧭</span>
      <h3>平台简介</h3>
      <p>了解 Zenith 解决什么问题、适合谁使用，以及统一入口不等于自动调度等关键边界。</p>
    </a>
    <a class="entry-card" href="./resources/overview.html">
      <span class="entry-card-icon">🧩</span>
      <h3>资源说明</h3>
      <p>认识资源区、队列、规格、开发机、镜像、存储、项目和配额之间的关系。</p>
    </a>
    <a class="entry-card" href="./guide/quick-start.html">
      <span class="entry-card-icon">🚀</span>
      <h3>五分钟快速开始</h3>
      <p>完成登录、检查资源、创建第一台开发机、等待就绪并选择合适的连接方式。</p>
    </a>
    <a class="entry-card" href="./compute/instances.html">
      <span class="entry-card-icon">🖥️</span>
      <h3>新建与管理开发机</h3>
      <p>了解快速新建和按资源区规格新建的向导，并安全执行启动、停止、重启、调整规格与删除。</p>
    </a>
    <a class="entry-card" href="./workloads/training.html">
      <span class="entry-card-icon">⚡</span>
      <h3>训练与推理</h3>
      <p>配置训练任务、查看运行状态，并判断当前环境是否开放推理服务能力。</p>
    </a>
    <a class="entry-card" href="./data/file-migrations.html">
      <span class="entry-card-icon">📦</span>
      <h3>数据流转</h3>
      <p>浏览存储，了解平台管理员的文件分享入口，并在满足挂载、所有权和操作权限时迁移数据。</p>
    </a>
    <a class="entry-card" href="./projects/overview.html">
      <span class="entry-card-icon">👥</span>
      <h3>项目与用量</h3>
      <p>管理成员和模板，理解开发机的项目标签、配额、卡时额度与费用估算方式。</p>
    </a>
    <a class="entry-card" href="./guide/cli-workflows.html">
      <span class="entry-card-icon">⌨️</span>
      <h3>CLI 与 Agent Skills</h3>
      <p>安装个人命令行工具，选择最小权限，查询资源、操作开发机并恢复未决回执。</p>
    </a>
    <a class="entry-card" href="./troubleshooting/common.html">
      <span class="entry-card-icon">🛟</span>
      <h3>问题排查</h3>
      <p>从权限、资源区连接、资源状态、SSH、终端、任务和迁移提示中找到下一步。</p>
    </a>
  </div>

  <div class="scope-note">
    <strong>适用范围：</strong>本文档以 Zenith 当前控制台为准，覆盖普通用户、项目管理员、运维和平台管理员的主要可见功能。具体入口和可执行操作仍取决于平台角色、项目角色、资源区连接状态、区域资源和平台管理员配置；页面提示是当前环境的最终依据。
  </div>
</div>
