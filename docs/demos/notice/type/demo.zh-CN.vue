<template>
  <div>
    <Button type="primary" @click="$notice.info('提示的标题', '一条一般提示消息')">
      一般提示
    </Button>
    <Button type="success" @click="$notice.success('提示的标题', '一条成功提示消息')">
      成功提示
    </Button>
    <Button type="warning" @click="$notice.warning('提示的标题', '一条警告提示消息')">
      警告提示
    </Button>
    <Button type="error" @click="$notice.error('提示的标题', '一条错误提示消息')">
      错误提示
    </Button>
  </div>
</template>
